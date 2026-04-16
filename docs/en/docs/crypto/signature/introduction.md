# Digital Signatures

In daily life, when we participate in certain activities, we may need to sign our names to prove that we were indeed present — for example, to prevent the class advisor from marking us absent... you know how it is. But in reality, such signatures are easily forged — just find someone to sign on your behalf, or find someone who can imitate another person's handwriting. In the digital world, we may need electronic signatures because we mostly deal with electronic documents. So what do we do? Of course, we can still choose to use our own name. But there is another approach: using digital signatures. These are much harder to forge and far more trustworthy. The main purpose of digital signatures is to ensure that a message truly comes from the person who claims to have produced it.

Digital signatures are primarily used to sign digital messages to prevent forgery or tampering of messages under a false identity, and can also be used for mutual identity authentication between communicating parties.

Digital signatures rely on asymmetric cryptography, because we must ensure that one party can do something that the other party cannot. The basic principle is as follows:

![](./figure/Digital_Signature_diagram.png)

Digital signatures should have the following properties:

(1) Signatures are trustworthy: Anyone can verify the validity of a signature.

(2) Signatures are unforgeable: It is difficult for anyone other than the legitimate signer to forge their signature.

(3) Signatures are non-transferable: A signature on one message cannot be copied to become a signature on another message. If a signature on a message was copied from elsewhere, anyone can detect the inconsistency between the message and the signature, and thus reject the signed message.

(4) Signed messages are unalterable: A signed message cannot be tampered with. Once a signed message has been tampered with, anyone can detect the inconsistency between the message and the signature.

(5) Signatures are non-repudiable: The signer cannot later deny their own signature.
