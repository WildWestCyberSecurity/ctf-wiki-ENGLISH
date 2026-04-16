# Summary of Offline Attack-Defense Experience


First, a normal competition will provide an interface for submitting flags, with the interface address similar to `http://172.16.4.1/Common/submitAnswer`. Generally, we need to submit flags through the interface according to the documentation provided by the organizer. In competitions, the interface address uses the Post method for submission, with two parameters: one is `Answer`, whose value is the obtained flag string, and the other is `token`, whose value is each team's team Token.

The organizer will also provide each participating team with a **virtual machine for analyzing network traffic**. Contestants need to access the designated address to download traffic files for analysis.

## Monitor Gamebox Status

During the competition, you can check the status of both your own and the opposing teams' GameBoxes. Keeping a close watch allows you to obtain competition information early and make adjustments accordingly.

For your own GameBox, there are several reasons that may cause it to go down:

1.  The organizer's judging system makes an error, incorrectly judging the GameBox as unavailable. This situation can usually be discovered before the competition starts. If you notice this, alert the staff as soon as possible to minimize losses.
2.  A faulty program patch causes the service to become unavailable. After patching a program, pay attention to the GameBox status in the next round. If a patch error causes unavailability, recover promptly. However, don't be overly worried about reverting to the original unpatched vulnerable program. Being down results in very few points split among all teams, whereas running the vulnerable program allows strong teams to exploit it for high scores. Therefore, handle each situation accordingly.
3.  An opponent's improper attack causes the GameBox to become unavailable. If discovered, recover promptly.
4.  The organizer strengthens program checks. In this case, the organizer will notify all participants. On the GameBox status wall, you will see widespread unavailability of that challenge's GameBoxes across teams.

For opposing teams' GameBoxes, we can obtain the following information:

1.  By observing attack traffic, identify which teams' GameBoxes have not successfully defended. Focus more attacks on those teams.
2.  When a team achieves the first blood, you can infer from each team's GameBox status whether the first-blood team has already written an exploit script. After writing an exploit script, observe whether your own defenses are in place.

## Distinguish Network Segments and Ports

During the competition, the organizer will arrange a reasonable network segment distribution.

When maintaining, you need to connect to the network segment where your team's GameBox is located and log in using the CTF account and password provided by the organizer. When interacting with other teams' GameBoxes, you need to connect to the corresponding network segment to interact with the vulnerable programs. Submitting flags requires going to the designated challenge platform.

!!! warning
    Pay special attention to ports here. If a port is accidentally misconfigured, such an error can be quite difficult to detect, and this kind of mistake can lead to unnecessary losses. It can even result in the critical situation of being unable to submit flags for an extended period. So be careful.

## Service Patching and Defense

1.  Program patches should be reasonable and comply with the judging system's check conditions. Although what exactly the system checks is not publicly disclosed, under normal circumstances, the system will not be overly strict.
2.  Use IDA for program patching. IDA provides three patching methods:
    byte, word, and assemble. Among these, byte-level patching is quite useful. Since modifying byte by byte doesn't require considering assembly instructions, such modifications are typically small and very practical in certain scenarios. Assembly instruction-level modification, while convenient since it doesn't require modifying bytecode, also introduces certain difficulties — for example, you need to additionally consider the length of assembly instructions, whether the structure is reasonable and complete, whether the logic remains the same as the original, and whether the modified assembly instructions are valid.
3.  When patching programs, remember to back up the original vulnerable program for team analysis. When uploading the patch, you should first delete the original vulnerable program, then copy the patched program in. After copying, you also need to assign the appropriate permissions to the program.
4.  In a typical competition, there will be over a dozen places in the vulnerable program that need patching. When patching, not only should the patches be effective and reasonable, but they should also be able to prevent or obfuscate opponents' analysis to some extent.

## Build a Script Framework for Rapid Attack Deployment

In attack-defense competitions, achieving first blood is especially important. Therefore, having an attack script framework is very advantageous. Rapidly developing attack scripts allows you to maintain an advantage in the early stages, and also saves time to work on defenses while continuously scoring.

## Competition Strategies

1.  During the competition, don't get stuck on a single challenge. Given the advantage of first blood, you should get an overall understanding of the difficulty of all challenges and start analyzing from the **easier ones**, progressing step by step.
2.  During the competition, the gap between teams will widen significantly. You should focus on attacking teams of similar strength and those stronger than yours, especially when scores are close — tighten both your attack and defense.
3.  During the competition, NPCs will periodically send attack traffic. You can obtain payloads from this attack traffic.
4.  Make sure to attack the NPCs aggressively.
5.  At the start of the competition, you can set all management passwords to the same password for easier team member login and management. In the initial phase, back up all files for sharing within the team.
