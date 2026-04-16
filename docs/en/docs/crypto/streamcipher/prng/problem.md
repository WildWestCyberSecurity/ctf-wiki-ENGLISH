# Challenges

## 2017 Tokyo Westerns CTF 3rd Backpacker's Problem

The challenge provides a cpp file:

```cpp
#include <iostream>
#include <vector>
#include <cstdint>
#include <random>
#include <algorithm>
#include <signal.h>
#include <unistd.h>

typedef __int128 int128_t;

std::istream &operator>>(std::istream &is, int128_t &x) {
    std::string s;
    is >> s;
    bool neg = false;
    if (s.size() > 0 && s[0] == '-') {
        neg = true;
        s = s.substr(1);
    }
    x = 0;
    for (char t : s)
        x = x * 10 + t - '0';
    if (neg)
        x = -x;
    return is;
}

std::ostream &operator<<(std::ostream &os, int128_t x) {
    if (x < 0)
        return os << "-" << (-x);
    else if (x == 0)
        return os << "0";
    else {
        std::string s = "";
        while (x > 0) {
            s = static_cast<char>('0' + x % 10) + s;
            x /= 10;
        }
        return os << s;
    }
}

static std::mt19937 mt = std::mt19937(std::random_device()());
int128_t rand(int bits) {
    int128_t r = 0;
    for (int i = 0; i < bits; i += 32) {
        r = (r << 32) | mt();
        if (i + 32 > bits) {
            r >>= (i + 32 - bits);
        }
    }
    if (mt() & 1) {
        r = -r;
    }
    return r;
}

std::vector<int128_t> generate_problem(int problem_no) {
    int n = problem_no * 10;
    int m = n / 2;
    while (true) {
        std::vector<int128_t> ret;
        int128_t tmp = 0;
        for (int i = 0; i < m - 1; i++) {
            ret.push_back(rand(100));
            tmp -= ret.back();
        }
        if (tmp < 0 && ((-tmp) >> 100) > 0)
            continue;
        if (tmp > 0 && (tmp >> 100) > 0)
            continue;
        ret.push_back(tmp);
        for (int i = m; i < n; i++) {
            ret.push_back(rand(100));
        }
        std::sort(ret.begin(), ret.end());
        ret.erase(std::unique(ret.begin(), ret.end()), ret.end());
        return ret;
    }
}

const int P = 20;
const int TLE = 4;

void tle(int singals) {
    std::cout << "Time Limit Exceeded" << std::endl;
    std::exit(0);
}

bool check(bool condition) {
    if (!condition) {
        std::cout << "Wrong Answer" << std::endl;
        std::exit(0);
    }
    return true;
}

int main() {
    signal(SIGALRM, tle);
    std::cout << R"(--- Backpacker's Problem ---
Given the integers a_1, a_2, ..., a_N, your task is to find a subsequence b of a
where b_1 + b_2 + ... + b_K = 0.

Input Format: N a_1 a_2 ... a_N
Answer Format: K b_1 b_2 ... b_K

Example Input:
4 -8 -2 3 5
Example Answer:
3 -8 3 5
)" << std::endl;
    for (int problems = 1; problems <= P; problems++) {
        std::cout << "[Problem " << problems << "]" << std::endl;
        std::cout << "Input: " << std::endl;
        std::vector<int128_t> in = generate_problem(problems);
        std::cout << in.size();
        for (auto t : in)
            std::cout << " " << t;
        std::cout << std::endl;
        alarm(TLE);
        std::cout << "Your Answer: " << std::endl;
        int K;
        std::cin >> K;
        // Check Size
        check(K > 0 && K < static_cast<int>(in.size()));
        std::vector<int128_t> b(K);
        for (int i = 0; i < K; i++)
            std::cin >> b[i];
        alarm(0);
        // Check Subsequence
        check(std::is_sorted(b.begin(), b.end()));
        check(std::unique(b.begin(), b.end()) == b.end());
        check(std::remove_if(b.begin(), b.end(), [&](int128_t k) -> bool {
                  return !binary_search(in.begin(), in.end(), k);
              }) == b.end());
        // Check sum
        check(!std::accumulate(b.begin(), b.end(), 0));
    }
    // Give you the flag
    // std::cout << std::getenv("FLAG") << std::endl;
    std::cout << "flag{test}" << std::endl;
    return 0;
}
```

We can see that, like competitive programming problems, it provides a problem statement with input/output examples:

```
Given the integers a_1, a_2, ..., a_N, your task is to find a subsequence b of a
where b_1 + b_2 + ... + b_K = 0.

Input Format: N a_1 a_2 ... a_N
Answer Format: K b_1 b_2 ... b_K

Example Input:
4 -8 -2 3 5
Example Answer:
3 -8 3 5
```

This is a subset sum (knapsack) problem. In this challenge, we need to solve $20$ such knapsack problems, with knapsack sizes ranging from 1 * 10 to 20 * 10. The subset sum knapsack problem is an NPC problem, and the time complexity grows exponentially with the knapsack size. Here the maximum knapsack size is $200$, so obviously brute force is not feasible.

There are two solution approaches: divide and conquer and lattice. Here we discuss

Divide and conquer strategy:

For the case where $n > 20$, decompose the problem into a subset sum problem of the first $20$ elements and the remaining elements, using a hash table to quickly find qualifying subsets.

For this challenge, during the table building and lookup process, a useful trick is to prefer `std::unordered_map` over `std::map`, as the performance is better (average time complexity of $O(1)$).

By looking up the documentation for `std::accumulate`, we know that the type of init determines the type of the accumulated result. If init is int, then the accumulated result is also int; if init is double, then the accumulated result is also double.

Here the initial value is `0`, so the space of the accumulated result is only int size, meaning we only need to read the lower $32$ bits when reading data.

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>

using namespace std;

// 计算子集和的辅助函数
void find_subset_sum(const vector<unsigned int> &A, vector<int> &ans) {
    const int n = A.size();
    if (n <= 20) {
        // 如果 n <= 20，直接暴力枚举所有子集
        for (int b = 1; b < (1 << n); ++b) {
            unsigned int S = 0;
            for (int i = 0; i < n; ++i) {
                if (b >> i & 1) {
                    S += A[i];
                }
            }
            if (S == 0) {
                for (int i = 0; i < n; ++i) {
                    if (b >> i & 1) {
                        ans.push_back(i);
                    }
                }
                return;
            }
        }
    } else {
        // 如果 n > 20，使用分治策略
        unordered_map<unsigned int, int> M; // 存储前20个元素的子集和
        const int k = 20;                  // 分治的前20个元素

        // 枚举前20个元素的子集
        for (int b = 0; b < (1 << k); ++b) {
            unsigned int S = 0;
            for (int i = 0; i < k; ++i) {
                if (b >> i & 1) {
                    S += A[i];
                }
            }
            M[-S] = b; // 存储 -S 和对应的子集
        }

        // 枚举剩余元素的子集
        for (int b = 0;; ++b) {
            unsigned int S = 0;
            for (int i = k; i < n; ++i) {
                if (b >> (i - k) & 1) {
                    S += A[i];
                }
            }

            // 查找是否存在前20个元素的子集与当前子集的和为0
            if (auto it = M.find(S); it != M.end() && (b != 0 || it->second != 0)) {
                const int prefix_mask = it->second;
                for (int i = 0; i < k; ++i) {
                    if (prefix_mask >> i & 1) {
                        ans.push_back(i);
                    }
                }
                for (int i = k; i < n; ++i) {
                    if (b >> (i - k) & 1) {
                        ans.push_back(i);
                    }
                }
                return;
            }
        }
    }
}

int main() {
    int n;
    cin >> n;
    if (n <= 0) {
        cout << 0 << endl;
        return 0;
    }

    vector<unsigned int> A(n);
    for (unsigned int &a : A) {
        cin >> a;
    }

    vector<int> ans;
    find_subset_sum(A, ans);

    cout << ans.size();
    for (int a : ans) {
        cout << " " << a;
    }
    cout << endl;

    return 0;
}
```

We have solved a single query. Using pwntools makes it more convenient to provide interactive capabilities.

```python
# 像是 OI 的交互题
from pwn import context, process

# 连接到本地服务器
context.log_level = 'debug'  # 启用调试日志
io1 = process("/backpacker-server/server")

def readline():
    return io1.recvline().decode().strip()

# 读取前12行输出 (case)
for _ in range(12):
    readline()

# 处理接下来的20轮交互
for _ in range(20):
    readline()
    readline()
    input_list = readline()
    readline()

    # 处理输入数据
    input_list = list(map(int, input_list.split()))  # 将输入数据转换为整数列表

    # lambda 函数将列表中的每个元素与 0xffffffff 按位与
    # map 函数将 lambda 函数应用到列表中的每个元素
    # join 函数将列表中的元素连接成字符串，用空格分隔
    input_cut = " ".join(map(lambda x: str(x & 0xffffffff), input_list))
    print(f'input: {input_cut}')

    # 使用 pwntools 的 process 调用外部程序
    # 编译题目提供的 server.cpp 文件, 方便交互
    # g++ -std=c++11 server.cpp -o server
    io2 = process("./server")
    io2.sendline(input_cut.encode())
    input_solve = io2.recvline().decode().strip()

    input_solve = list(map(int, input_solve.split()))
    first_element = str(input_solve[0])
    
    remaining_elements = []
    for x in input_solve[1:]:
        element = str(input_list[x + 1])
        remaining_elements.append(element)
    
    input_solve = " ".join([first_element] + remaining_elements)
    io1.sendline(input_solve.encode())

    io2.close()

print(readline())

io1.close()
```


## References

-   https://github.com/r00ta/myWriteUps/tree/master/GoogleCTF/woodman
-   http://mslc.ctf.su/wp/google-ctf-woodman-crypto-100/
-   https://github.com/ymgve/ctf-writeups/tree/master/tokyowesterns2017/ppc-backpackers_problem
-   https://en.cppreference.com/w/cpp/algorithm/accumulate
-   https://qiita.com/kusano_k/items/b1fff79d535f4b26cdd0#backpackers-problem
