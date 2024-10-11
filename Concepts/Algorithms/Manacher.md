## Manacher 알고리즘

가장 긴 팰린드롬의 길이를 구하는 문제는 브루트포스로 O(N^3), DP로 O(N^2)에 풀이가 가능하다.

하지만 조금 더 최적화를 하면 O(N)에 가능한데, 이를 가능하게 하는 것이 Manacher 알고리즘이다.

Manacher 알고리즘의 핵심 포인트는 다음과 같다.

> 이전에 구했던 회문 중 가장 오른쪽으로 긴 회문의 중심 인덱스 `p` 와 오른쪽 끝 인덱스인 `r` 값을 통해 이전에 구한 데이터를 활용하여 연산을 최적화

<br/>

### 초기값

<img src="../imgs/Pasted image 20241010180856.png" />

<br/>

- `p` : 이전까지 구한 팰린드롬 중 가장 긴 팰린드롬의 중심 인덱스
- `r` : 이전까지 구한 팰린드롬 중 가장 긴 팰린드롬의 오른쪽 끝 인덱스
- `l` : p를 중심으로 한 r의 대칭점

위 변수의 관계를 나타내면 다음과 같다.

```c++
p = (l + r) / 2
```

- `s[i]` : 문자열에서 i번 인덱스의 문자
- `res[i]` : i번 문자를 중심으로 했을 때, 팰린드롬 최대 길이


<br/>

### 최적화 과정

최적화를 위해서는 두 번의 비교문이 필요하다.

#### 순차적으로 돌고 있는 i를 기준으로 비교

1. `i` < `r` 인 경우
2. `i` ≥ `r` 인 경우 → 이전에 구한 정보(res[j])를 활용할 수 없어서, 직접 구해야 한다.

처음 시작할 때 or 현재 탐색하는 위치 `i` 가 `r` 보다 오른쪽에 있는 경우에는 이전에 구한 정보(res[j])를 활용할 수 없다.

`i` 와 `r` 이 같을 때는 


#### 1번이라면? res[j]와 (i-r)을 비교



##### 1. `res[j] < i-r` : `res[i] = res[j]`

<img src="../imgs/Pasted image 20241010181943.png" />

여기서 j의 값은 i보다 작으므로 

##### 2. `res[j] > i-r`

<img src="../imgs/Pasted image 20241010181956.png" />

`res[j]` 가 `i-r` 보다 클 때, `res[j]` 를 활용할 수 없다.

- S[l-1]과 j를 기준으로 대칭점인 x는 j를 중심으로 한 팰린드롬 안에 있으므로 같다.
- 또한 x와 p를 기준으로 대칭점인 y는 p를 중심으로 한 팰린드롬 안에 있으므로 같다.

- 위 두 가지에 의해 y와 r을 기준으로 대칭점인 z(r+1)은 p를 중심으로 한 대칭점이 S[l-1]이어야 한다.

하지만 p를 중심으로 한 최대 길이 팰린드롬의 끝이 `r` 이어야 하기 때문에, 이는 



## 코드

```java
public static int[] manacher(String s) {
    int n = s.length();
    int[] res = new int[2 * n + 1];
    StringBuilder t = new StringBuilder("#");

    for (char c : s.toCharArray()) {
        t.append(c).append("#"); // 홀수와 짝수 상관 없이 탐색하기 위해 넣어줌
    }

    // 1. i를 기준으로 순차적으로 탐색
    int p = 0, r = 0;
    for (int i = 0; i < t.length(); i++) {
        int j = 2 * p - i; // p를 중심으로 할 때, i의 대칭점

        if (i < r) { // 이전에 구한 값 사용 가능
            res[i] = Math.min(r - i, res[j]);
        }
        while (i - 1 >= res[i]
                && i + res[i] + 1 < t.length()
                && t.charAt(i - res[i] - 1) == t.charAt(i + res[i] + 1)) {
            res[i]++;
        }

        if (res[i] > r - i) {
            p = i;
            r = i + res[i];
        }
    }

    // 2. 결과를 원래 문자열의 길이에 맞게 변환
    int[] maxLength = new int[n];

    for (int i = 0; i < n; i++) {
        maxLength[i] = res[2 * i + 1];
    }
    return maxLength;
}
```



## 참고

- [Manacher 알고리즘 - IOI 영상](https://www.youtube.com/watch?v=OLpy_Mh6NZY)
- [Manacher 알고리즘 - 블로그 (그림 출처)](https://ialy1595.github.io/post/manacher/)
