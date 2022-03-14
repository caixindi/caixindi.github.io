> 快速傅里叶变换(FFT)

​	FFT的任务就是将多项式的系数表示法转化为点值表示法，再由点值表示法转化为系数表示法的过程。前面的过程称为求值（DFT），后面的过程称为插值（IDFT）。

​	任取n+1个互不相同的$S=\lbrace p_1,p_2,...,p_{n+1}\rbrace$,对$f(x)$分别求值后得到$f(p_1),f(p_2),...,f(p_{n+1})$，此时称$A(x)=\lbrace(p_1,f(p_1)),(p_2,f(p_2)),...,(p_{n+1},f(p_{n+1}))\rbrace$为多项式f(x)在S下的点值表示法。

​	得到一个n次多项式的点值表示法需要代入n+1个数到多项式里面去，如果只是随意选取n+1个，每个数值的计算复杂度为O(n)，那么n+1个数值的计算复杂度将仍然是O(n²)，因此想到了利用奇偶函数的对称性代入，这样计算量就变为了原来的一半，之后不断递归，但由于后续的递归过程如果代入的是实数平方，后续递归将不能进行。由此便引入了复数。通过将n次单位根组成S，记$A_0(x)$为偶次项和，$A_1(x)$为奇次项和：
$$
A_0(x)=a_0x^0+a_2x^1+...+a_{n-1}x^{\frac{n}{2}}
$$
$$
A_1(x)=a_1x^0+a_3x^1+...+a_{n-2}x^{\frac{n}{2}}
$$

于是有$A(\omega_n^m)=a_0\omega_n^0+a_1\omega_n^m+a_2\omega_n^{2m}+a_3\omega_n^{3m}+...+a_{n-1}\omega_n^{(n-1)*m}+a_n\omega_n^{nm}$

之后：$A(\omega_n^m)=A_0((\omega_n^m)^2)+\omega_n^mA_1((\omega_n^m)^2)=A_0(\omega_\frac{n}{2}^m)+\omega_n^mA_1(\omega_\frac{n}{2}^m)$

$A(\omega_n^{m+\frac{n}{2}})=A_0((\omega_n^m)^2)+\omega_n^{m+\frac{n}{2}}A_1((\omega_n^m)^2)=A_0(\omega_\frac{n}{2}^m)-\omega_n^mA_1(\omega_\frac{n}{2}^m)$

因此递归的FFT伪代码为：

```python
def FFT(A)
# A-[a0,a1,...,an-1]
n=len(A)
if n==1:
    return A
A0,A1=[a0,a2,...,an-2],[a1,a3,...,an-1]
a=[0]*n
for i in range(n/2):
     a[i]=a0[i]+w*a1[i];
     a[i+n/2]=a0[i]-w*a1[i]
     w=w*wn
return a
```

