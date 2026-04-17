# CGC Super Challenge

> This section is excerpted from Professor Li Kang's speech "Vulnerability Discovery and Exploitation in the Cyber Grand Challenge" at the ISC Internet Security Conference on August 17, 2016.

The CGC (Cyber Grand Challenge) is the world's first fully automated machine-based cyber attack and defense competition, with the entire process being fully automated without any human intervention. It tests machines' capabilities in automatic vulnerability discovery, automatic software hardening, automatic vulnerability exploitation, and automatic network defense. It uses a simplified Linux operating system — DECREE — and a Snort-like rule-based filtering firewall. Vulnerability discovery is performed on Linux binary programs. None of the participating teams have access to the program source code.

In the 2016 CGC competition, the challenge problems encompassed 53 types of CWEs, including 28 heap overflow vulnerabilities, 24 stack overflow vulnerabilities, 16 null pointer dereference vulnerabilities, 13 integer overflow vulnerabilities, and 8 Use-After-Free (UAF) vulnerabilities.

The attack and defense process works as follows: the organizer releases challenge programs, and each team's server can submit patched programs, firewall rules, and exploit programs to the organizer. The patched programs and firewall rules are then distributed to other teams. The organizer runs the challenge programs for each team, conducts service testing and attacks, and performs evaluations.

## Performance Evaluation Metrics

1.  Response time for normal service access;
2.  Patching frequency;
3.  Efficiency of hardened programs;
4.  Statistics on successful defense against attacks;
5.  Statistics on successful attacks.

## Defining the Core Task

Receive the binary program, perform automatic analysis, then harden the program and generate exploit programs after defining the firewall rules.

## Analysis Methods

1.  Concrete Execution - using normal execution mode;
2.  Symbolic Execution - assisting path selection during the Fuzzing phase;
3.  Concolic Execution - symbolic execution with concrete inputs, selecting paths based on inputs while preserving symbolic conditions.

## Lessons Learned from CGC

1.  Achieving a perfect defense is far more difficult than generating attacks;
2.  Binary hardening must avoid loss of functionality and minimize performance overhead;
3.  The trend toward automated security processing has taken shape — most teams can generate attacks and effective defenses for simple applications within seconds;
4.  Strategies in adversarial competition are worth studying — resources and actions should be adjusted based on your own and your opponent's attack and defense capabilities.
