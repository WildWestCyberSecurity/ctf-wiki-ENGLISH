# CFB

CFB stands for Cipher Feedback mode.

## Encryption

![](./figure/cfb_encryption.png)

## Decryption

![](./figure/cfb_decryption.png)

## Advantages and Disadvantages

### Advantages

- Adapts to different data format requirements
- Limited error propagation
- Self-synchronizing

### Disadvantages

- Encryption cannot be parallelized, decryption cannot be parallelized

## Application Scenarios

This mode is suitable for encryption environments with special data format requirements, such as database encryption and wireless communication encryption.

## Challenges

- HITCONCTF-Quals-2015-Simple-(Crypto-100)

