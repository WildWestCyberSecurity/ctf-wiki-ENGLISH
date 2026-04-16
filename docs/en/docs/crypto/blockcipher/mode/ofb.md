# OFB

OFB stands for Output Feedback mode. Its feedback content is the output of the block encryption rather than the ciphertext.

## Encryption

![](./figure/ofb_encryption.png)

## Decryption

![](./figure/ofb_decryption.png)

## Advantages and Disadvantages

### Advantages

1. Does not have error propagation properties.

### Disadvantages

1. The IV does not need to be kept secret, but a different IV must be chosen for each message.
2. Does not have self-synchronization capability.

## Applicable Scenarios

Suitable for scenarios where the plaintext has a high degree of redundancy, such as image encryption and voice encryption.
