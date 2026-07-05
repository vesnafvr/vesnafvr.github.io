+++
date = '2026-07-01T08:00:00-05:00'
draft = false
title = "Reviewing MulVAL: reading the ancestor of a tool I'm building"
+++

Today, cloud environments often have misconfigurations that let attackers pivot between roles, escalate admin privileges, and pivot within account boundaries. These are difficult to identify because individual misconfigurations look benign in isolation but become dangerous when chained across multiple services. As a result, the attack surface becomes too large to enumerate by hand. Currently, I am working on a project that aims to identify these misconfigurations with a neurosymbolic engine using Datalog with a reinforcement learning agent that discovers and ranks the escalation chains an attacker would find first. I will be presenting this tool, called PathQuarry, at Summercon 2026.

![MulVAL attack graph](/images/mulval-overview.png)

In this post, I am reviewing "MulVAL: A Logic-based Network Security Analyzer", a USENIX paper written by Xinming Ou, Sudhakar Govindavajhala, and Andrew W. Appel, all from Princeton University. MulVAL is a reasoning framework that generates attack graphs for enterprise-scale networks. I'm reading this as someone building the descendant of this tool, so I came in half-expecting network-era assumptions to break with cloud era assumptions, but instead I was surprised by how similarly the two layers behave. MulVAL and PathQuarry are not solving the same problem, to be clear. MulVAL questions the completeness of a network in terms of whether any attack path can violate the security policy while PathQuarry asks how different privilege-escalation paths are prioritized as they are enumerated.

MulVAL's reasoning engine uses Datalog rules that map a network's behavior and interactions of various components of the system. It integrates findings from reported bugs and scans as well. As information is collected, analysis is performed and formulates an attack graph of the system. The scope of MulVAL comprises reported vulnerabilities, hosts and their configurations, routers, firewalls, users, and relevantly, access permissions. MulVAL is used as a scanner first, and then it runs an analyzer, run on one host as new information is collected from the scanner.

The analyzer can reason over transitive relationships to discover how an attacker can leverage an exploit as opposed to identifying that an attacker could leverage an exploit. This is where Datalog becomes very useful. Datalog is a declarative logic programming language that is used as a query language. It is a subset of Prolog, another well known logic programming language. It structures data as Facts and Rules, where Facts define existing data and Rules define logic that derives new data. The syntax is read like if-then statements in a Head :- Body format. In program analysis, many security tools and compilers use Datalog engines such as Soufflé to detect software vulnerabilities.

MulVAL uses this to show that if P can execute code on H as Owner, P gains arbitrary access to Owner's files:

```prolog
accessFile(P, H, Access, Path) :-
    execCode(P, H, Owner),
    filePath(H, Owner, Path).
```

In contrast, if the attacker can modify Owner's files, the attacker can gain Owner's privileges due to a Trojan horse that "can be injected by modified execution binaries, which Owner might then execute":

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

This paper has conceptual influence on my work. Cloud security environments resemble those of network security environments. Both environments consist of "thousands of machines", where reachability within a graph is important from a privilege escalation and declarative policy point of view. Reading MulVAL, I kept seeing similarities between the two. Hosts map to resources and CVEs to over-permissive IAM policies, and the multi-host network attack it models is recognizably a cloud privilege-escalation path — which is precisely why a Datalog reasoning engine smoothly moves between the two worlds, but I will get into that later.

However, the differences between the two worlds present a gap to me. Within MulVAL, the dangerous primitive is a broken software bug. In cloud privesc, the dangerous primitive is something that works exactly as designed. The exploit is not necessarily a memory bug but rather the composition of a policy. In the context of the cloud, MulVAL's network reachability carries over to PathQuarry, but with the addition of another layer: identity reachability. Network reachability describes whether one host can reach another through security groups, routes, ports, etc. while identity reachability describes whether one principal can act as another through the permission graph via policy chains, and cloud privesc generally acts within the boundaries of identity reachability. To summarize the analogy, a cloud IAM attack path is a MulVAL attack graph where the network has been replaced by the permission graph and the vulnerabilities by misconfigurations. Within the paper, I liked the sysadmin mention because it forced me to reflect upon who exactly my tool is for: the cloud operator that controls cloud permissions with an API key. A vulnerability a 2005 sysadmin was too slow to patch perhaps equates to the cloud operator of today writing a policy that was too broad and was exploited. 

Also, a glaring difference between MulVAL and PathQuarry is the sheer graph size comparison. Graphically, attack privileges accumulate as node count increases, so an attack graph would grow polynomially in input size. This also holds true for a cloud environment. But more importantly, the difference between the two is how edges are represented. In MulVAL, edges are vulnerabilities that are relatively sparse because most host pairs have no exploit edge between them, creating a graph that is large but sparse. In PathQuarry, edges come from permissions working as designed, and they grow dense because a single overly-permissive role can reach many other principals and resources. The difference in node count isn't significant but the edge density is, because it has implications about which permissions to prioritize during enumeration. 

In terms of scalability, MulVAL assumes that attacker privileges accumulate and facts are not retracted and PathQuarry assumes a role wouldn't cost starting privileges either. But as the graph becomes polynomial in input size, the significance of the resulting attack graph becomes increasingly pertinent. For MulVAL, the graph ends up being the deliverable but for a cloud environment, the resulting graph would be too big for a cloud operator to act on or even read, which is a gap that I am trying to solve with PathQuarry by targeting prioritization over showing a possible existing attack path.
 
## Thoughts on the paper

MulVAL was published in 2005, a time when the bug-reporting culture in the security community was a bit different from what it is today. Vulnerabilities were then becoming standardized for machines to parse as CVE identifiers and NVD entries. MulVAL depends on external scanners and a vulnerability database for recognition while the attack semantics stay in its hand-written rules. 

Something I liked about MulVAL is that it did not assume that its expected vulnerabilities would be in a clean, easy-to-read format as its scanners gathered vulnerability data automatically. It accepted that for a sysadmin, the data they would have would be coarse or incomplete. In the cloud era, IAM policies are easier to query so PathQuarry's effect is specifiable from AWS's own policy semantics.

Another feature of MulVAL that I particularly liked was the "bugHyp" predicate feature, where a fake bug is introduced into the reasoning process to simulate a potential bug and reveal second-order weaknesses in the network. I like that it preemptively considers what would happen if just one more thing went wrong. This sort of reminds me conceptually of reinforcement learning, where an agent can explore transitions that aren't known to be exploitable yet, also a feature I'm building out in PathQuarry.

## The case for Datalog

Datalog is a versatile language, used not just in program analysis, but also in graph applications, distributed systems (as in MulVAL's case), and cloud security. It has been used inside Biscuit tokens to compute complex identity access policies in cloud computing, for example. Cloud computing took off in the mid-late 2000s, just as this paper was published in 2005. As a result, Datalog presents itself as a strong tool that aligns IAM permissions and policies well. IAM policy is already a declarative grant language (a set of statements about who may do what to which resource), so encoding it as Datalog facts avoids the bottleneck that MulVAL has with translating network information into data; it would be a matter of semantic encoding. Datalog natively computes privilege escalation because it derives the full set of capabilities from a starting principal and Soufflé allows this to scale.

## Conclusion

Reading MulVAL closely clarified what I'm actually keeping and what I'm adding. I'm keeping the central idea: that a declarative logic engine is the right substrate for reasoning about reachability and privilege over a large graph. What changes is the nature of the exploit and what I add is a layer that MulVAL was not built for at the time. PathQuarry has to prioritize rather than enumerate as MulVAL does, because an IAM account's sound transition space is too large to hand a cloud operator as a flat graph. That's why a learned policy decides which of the many valid paths are the ones worth surfacing first. The ancestor reasons about what is possible while my descendant has to reason about what matters.
