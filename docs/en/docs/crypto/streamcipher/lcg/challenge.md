# Challenges

## 2016 Google CTF woodman

The general idea of the program is a number guessing game. If you guess correctly several times in a row, you can obtain the flag. The core code for generating the corresponding numbers is as follows:

```python
class SecurePrng(object):
    def __init__(self):
        # generate seed with 64 bits of entropy
        self.p = 4646704883L
        self.x = random.randint(0, self.p)
        self.y = random.randint(0, self.p)

    def next(self):
        self.x = (2 * self.x + 3) % self.p
        self.y = (3 * self.y + 9) % self.p
        return (self.x ^ self.y)
```

Obviously here we can see that guessing the first two rounds correctly is relatively easy, after all the probability is 0.25. Once we correctly guess the first two rounds, we can use Z3 to solve for the initial x and y, and then we can smoothly guess the remaining values.

The specific script is as follows, however Z3 seems to have issues when solving this kind of problem...

Here we consider another approach: **enumerate from the lowest bit to the highest bit to obtain the value of x**. The reason this works is based on the following observations:

- a + b = c, the value of the i-th bit of c is only affected by the corresponding bit and lower bits of a and b. **Because when operating on the i-th bit, it can only receive carry values from lower bits.**
- a - b = c, the value of the i-th bit of c is only affected by the corresponding bit and lower bits of a and b. **Because when operating on the i-th bit, it can only borrow from lower bits.**
- a * b = c, the value of the i-th bit of c is only affected by the corresponding bit and lower bits of a and b. This is because multiplication can be viewed as repeated addition.
- a % b = c, the value of the i-th bit of c is only affected by the corresponding bit and lower bits of a and b. This is because modulo can be viewed as repeated subtraction.
- a ^ b = c, the value of the i-th bit of c is only affected by the corresponding bit of a and b. This is self-evident.

**Note: I personally feel this technique is very useful.**

Furthermore, we can easily determine that p has 33 bits. The specific exploitation approach is as follows:

1. First, obtain two correctly guessed values. The probability of this is 0.25.
2. Enumerate **the corresponding bits of x after the first iteration** from the lowest bit to the highest bit sequentially.
3. Calculate the second value based on the enumerated values. Only when the corresponding bit is correct can it be added to the candidate correct values. Note that due to the modulo operation, we need to enumerate how many times the subtraction was performed.
4. Additionally, in the final verification, we still need to ensure the corresponding values satisfy certain requirements, because we previously enumerated how many times the subtraction was performed.

The specific exploitation code is as follows:

```python
import os
import random
from itertools import product


class SecurePrng(object):
    def __init__(self, x=-1, y=-1):
        # generate seed with 64 bits of entropy
        self.p = 4646704883L  # 33bit
        if x == -1:
            self.x = random.randint(0, self.p)
        else:
            self.x = x
        if y == -1:
            self.y = random.randint(0, self.p)
        else:
            self.y = y

    def next(self):
        self.x = (2 * self.x + 3) % self.p
        self.y = (3 * self.y + 9) % self.p
        return (self.x ^ self.y)


def getbiti(num, idx):
    return bin(num)[-idx - 1:]


def main():
    sp = SecurePrng()
    targetx = sp.x
    targety = sp.y
    print "we would like to get x ", targetx
    print "we would like to get y ", targety

    # suppose we have already guess two number
    guess1 = sp.next()
    guess2 = sp.next()

    p = 4646704883

    # newx = tmpx*2+3-kx*p
    for kx, ky in product(range(3), range(4)):
        candidate = [[0]]
        # only 33 bit
        for i in range(33):
            #print 'idx ', i
            new_candidate = []
            for old, bit in product(candidate, range(2)):
                #print old, bit
                oldx = old[0]
                #oldy = old[1]
                tmpx = oldx | ((bit & 1) << i)
                #tmpy = oldy | ((bit / 2) << i)
                tmpy = tmpx ^ guess1
                newx = tmpx * 2 + 3 - kx * p + (1 << 40)
                newy = tmpy * 3 + 9 - ky * p + (1 << 40)
                tmp1 = newx ^ newy
                #print "tmpx:    ", bin(tmpx)
                #print "targetx: ", bin(targetx)
                #print "calculate:     ", bin(tmp1 + (1 << 40))
                #print "target guess2: ", bin(guess1 + (1 << 40))
                if getbiti(guess2 + (1 << 40), i) == getbiti(
                        tmp1 + (1 << 40), i):
                    if [tmpx] not in new_candidate:
                        #print "got one"
                        #print bin(tmpx)
                        #print bin(targetx)
                        #print bin(tmpy)
                        new_candidate.append([tmpx])
            candidate = new_candidate
            #print len(candidate)
            #print candidate
        print "candidate x for kx: ", kx, " ky ", ky
        for item in candidate:
            tmpx = candidate[0][0]
            tmpy = tmpx ^ guess1
            if tmpx >= p or tmpx >= p:
                continue
            mysp = SecurePrng(tmpx, tmpy)
            tmp1 = mysp.next()
            if tmp1 != guess2:
                continue
            print tmpx, tmpy
            print(targetx * 2 + 3) % p, (targety * 3 + 9) % p


if __name__ == "__main__":
    main()
```
