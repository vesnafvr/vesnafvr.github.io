+++
date = '2026-07-01T08:00:00-05:00'
draft = false
title = "Reviewing MulVAL: reading the ancestor of a tool I'm building"
+++

Currently, I am working on an attack graph vulnerability discovery project, focusing on cloud permissions written in Datalog. Long story short, the project involves creating a neurosymbolic tool that pairs a Q-learning agent with Soufflé Datalog checking to discover and prioritize realistic privilege-escalation attack paths in AWS IAM configurations—using the symbolically-derived sound transition space both as the agent's training data and as a per-step check, so the resulting paths are scalable to find yet provably valid by construction. I will be presenting this tool, called PathQuarry, at Summercon 2026.

![MulVAL attack graph](/images/mulval-overview.png)

In this post, I am reviewing "MulVAL: A Logic-based Network Security Analyzer", a USENIX paper written by Xinming Ou, Sudhakar Govindavajhala, and Andrew W. Appel, all from Princeton University. MulVAL is a reasoning framework that generates attack graphs for enterprise-count networks. I'm reading this as someone building the descendant of this tool, so I came in half-expecting network-era assumptions to break with cloud era assumptions, but instead I was surprised by how similarly the two layers behave. MulVAL's reasoning engine uses Datalog rules that map a network's behavior and interactions of various components of the system. It integrates findings from reported bugs and scans as well. As information is collected, analysis is performed and formulates an attack graph of the system. The scope of MulVAL comprises reported vulnerabilities, hosts and their configurations, routers, firewalls, users, and relevantly, access permissions. MulVAL is used as a scanner first, and then it runs an analyzer, run on one host as new information is collected from the scanner.

The analyzer can reason over transitive relationships to discover how an attacker can leverage an exploit as opposed to identifying that an attacker could leverage an exploit. This is where Datalog becomes very useful. Datalog is a declarative logic programming language that is used as a query language. It is a subset of Prolog, another well known logic programming language. It structures data as Facts and Rules, where Facts define existing data and Rules define logic that derives new data. The syntax is read like if-then statements in a Head :- Body format. In program analysis, many security tools and compilers use Datalog engines such as Soufflé to map variable pointers and detect software vulnerabilities.

In MulVAL, the example they use to show an attacker P's access of machine H with Owner's privilege where the attacker P can have arbitrary access to files owned by Owner:

```prolog
accessFile(P, H, Access, Path) :-
    execCode(P, H, Owner),
    filePath(H, Owner, Path).
```

In contrast, if the attacker can modify Owner's files, the attacker can gain privilege of Owner due to a Trojan horse that "can be injected by modified execution binaries, which Owner might then execute":

```prolog
execCode(Attacker, H, Owner) :-
    accessFile(Attacker, H, write, Path),
    filePath(H, Owner, Path),
    malicious(Attacker).
```

There are rules beyond this compromise propagation, such as exploit rules, network file system rules, and multihop network access. Datalog is also used in policy specifications and within the analysis algorithm itself, that simulates the attack and includes a policy checking phase. Ultimately, a violation is detected by the algorithm if an access violates the policy. When running the MulVAL scanner on an example small network used by 700 users, MulVAL found a violation due to existing vulnerabilities:

```prolog
vulExists(workStation, 'CAN-2004-0427', kernel).
vulExists(workStation, 'CAN-2004-0554', kernel).
vulExists(workStation, 'CAN-2004-0495', kernel).
vulExists(workStation, 'CVE-2002-1363', libpng).
```

Once the libpng vulnerability was patched, the policy no longer had policy violations because 3 of the vulnerabilities were local exploitable kernel bugs, which were hosted on machines only accessed by trusted users. The authors note these were trusted-user-only kernel bugs on machines whose months-long uptimes made a reboot costlier than the risk. Ultimately, the policy is defined by the system's own security goals.

## MulVAL's influence on PathQuarry

This paper has conceptual influence on my work. Cloud security environments resemble those of network security environments. Both environments consist of "thousands of machines", where reachability within a graph is important from a privilege escalation and declarative policy point of view. Reading MulVAL, I kept seeing similarities between the two. Hosts map to resources and CVEs to over-permissive IAM policies, and the multi-host network attack it models is recognizably a cloud privilege-escalation path — which is precisely why a Datalog reasoning engine smoothly moves between the two worlds but I will get into that later.

However, the differences between the two worlds present a gap to me. Within MulVAL, the dangerous primitive is a broken software bug. In cloud privesc, the dangerous primitive is something that works exactly as designed. The exploit is not necessarily a memory bug but rather the composition of a policy. Also, network reachability becomes a significant layer of the cloud environments, in addition to identity reachability. To summarize the analogy, a cloud IAM attack path is a MulVAL attack graph where the network has been replaced by the permission graph and the vulnerabilities by misconfigurations. Within the paper, I liked the sysadmin mention because it forced me to reflect upon who exactly my tool for: the cloud operator that controls cloud permissions with an API key. A patch that a sysadmin may have been too slow to update perhaps equates to a cloud operator writing a policy that was too broad and was exploited.

## Thoughts on the paper

MulVAL was published in 2005, a time when the bug-reporting culture in the security community was a bit different than it is today. Vulnerabilities were then becoming standardized for machines to parse as CVE identifiers and NVD entries. MulVAL depends on exploit preconditions being extracted from databases automatically, which outsources its threat model to community bug reports.

Another feature of MulVAL that I particularly liked was the "bugHyp" predicate feature, where a fake bug is introduced into the reasoning process to simulate a potential bug and reveal second-order weaknesses in the network. I like that it preemptively considers what would happen if just one more thing went wrong. This sort of reminds me conceptually of reinforcement learning, where an agent can explore transitions that aren't known to be exploitable yet, also a feature I'm building out in PathQuarry.

## The case for Datalog

Datalog is a versatile language, used not just in program analysis, but also in graph applications, distributed systems (as in MulVAL's case), and cloud security. It has been used inside Biscuit tokens to compute complex identity access policies in cloud computing, for example. Cloud computing took off in the mid-late 2000s, just as this paper was published in 2005. As a result, Datalog presents itself as a strong tool that aligns IAM permissions and policies well. IAM policy is already a declarative grant language (a set of statements about who may do what to which resource), so encoding it as Datalog facts does not have the bottleneck that MulVAL has, with translating network information into data; it would be a matter of semantic encoding. Datalog natively computes privilege escalation because it computes the full set of capabilities derived from a starting principal and Soufflé allows this to scale.

## Conclusion

Reading MulVAL closely clarified what I'm actually keeping and what I'm adding. I'm keeping the central idea: that a declarative logic engine is the right substrate for reasoning about reachability and privilege over a large graph. What changes is the nature of the exploit and what I add is a layer that MulVAL was not built for at the time. PathQuarry has to prioritize rather than enumerate as MulVAL does, because an IAM account's sound transition space is too large to hand a cloud operator as a flat graph. That's why a learned policy decides which of the many valid paths are the ones worth surfacing first. The ancestor reasons about what is possible while my descendant has to reason about what matters.
