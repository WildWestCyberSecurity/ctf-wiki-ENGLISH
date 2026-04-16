# Feedback Shift Register

In general, an n-stage feedback shift register is shown in the figure below:

![image-20180712201048987](./figure/n-fsr.png)

Where:

- $a_0$, $a_1$, …, $a_{n-1}$ are the initial state.
- F is the feedback function or feedback logic. If F is a linear function, it is called a Linear Feedback Shift Register (LFSR); otherwise, it is called a Nonlinear Feedback Shift Register (NFSR).
- $a_{i+n}=F(a_i,a_{i+1},...,a_{i+n-1})$.

In general, feedback shift registers are defined over a finite field to avoid numbers becoming too large or too small. Therefore, we can view it as a transformation within the same space, namely:

$(a_i,a_{i+1},...,a_{i+n-1}) \rightarrow (a_{i+1},...,a_{i+n-1},a_{i+n})$
.
For a sequence, we generally define its generating function as the sum of the power series corresponding to the sequence.
