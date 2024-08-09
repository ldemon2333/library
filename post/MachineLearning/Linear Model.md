# Supervised learning

​	To describe the supervised learning problem slightly more formally, our goal is, given a training set, to learn a function $h:\mathbf{x}\rightarrow \mathbf{y}$ so that $h(x)$ is a good predictor for the corresponding value of $y$.This function $h$ is called a **hypothesis**.

![image-20240623214437257](image-20240623214437257.png)

# Part I Linear Regression

​	As an initial choice, let's say we decide to approximate $y$ as a linear function of $x$:
$$
h_{\theta}(x) = \theta_0 + \theta_1x_1+\theta_2x_2
$$
 简化：这里假设$x_0=1$，$\theta_0$就是所谓的偏置bias.
$$
h(x)=\sum_{i=0}^{d}\theta_ix_i=\theta^Tx
$$
​	We define the **cost function**:
$$
J(\theta) = \frac{1}{2}\sum_{i=1}^n(h_{\theta}(x^{(i)})-y^{(i)})^2
$$

## 1 LMS algorithm

Choose $\theta$ so as to minimize $J(\theta)$.Use the gradient descent algorithm, which starts some initial $\theta$, and repeatedly performs the update:
$$
\theta_j:=\theta_j-\alpha\frac{\partial}{\partial\theta_j}J(\theta), j=0,...,d
$$
​	Let's first work it out for the case of if we have only one training example $(x,y)$, so that we can neglect the sum in the definition of $J$. We have:
$$
\begin{align}
\frac{\partial}{\partial\theta_j}J(\theta) &= \frac{\partial}{\partial\theta_j}\frac{1}{2}(h_{\theta}(x)-y)^2\\
&=2\cdot\frac{1}{2}(h_{\theta}(x)-y)\cdot\frac{\partial}{\partial\theta_j}(h_\theta(x)-y)\\
&=(h_\theta(x)-y)\cdot\frac{\partial}{\partial\theta_j}(\sum_{i=0}^d\theta_ix_i-y)\\
&=(h_\theta(x)-y)\cdot x_j
\end{align}
$$
​	For a single training example, this gives the update rule:
$$
\theta_j:=\theta_j +\alpha(y^{(i)}-h_{\theta}(x^{(i)}))\cdot x_j^{(i)}
$$
The rule is called the **LMS(least mean squares)** update rule and is also known as the **Widrow-Hoff** learning rule.
​	We'd derived the LMS rule for when there was only a single training example. There are two ways to modify this method for a training set of more than one example. The first is replace it with the following algorithm:

+  **Batch gradient descent**
---
​	
Repeat until convergence
$$
 	\theta_j:=\theta_j +\alpha \sum_{i=1}^n(y^{(i)}-h_{\theta}(x^{(i)}))\cdot x_j^{(i)}, j=0,...d\tag{1}
$$
​	By grouping the updates of the coordinates into an update of the vector $\theta$, we can rewrite update $(1)$ in a slightly more succinct way:
$$
\theta:=\theta +\alpha \sum_{i=1}^n(y^{(i)}-h_{\theta}(x^{(i)}))\cdot x^{(i)}
$$
​	This method looks at every example in the entire training set on every step, and is called **batch gradient descent.** Note that, while gradient descent can be susceptible to local minimum in general, the optimization problem we have posed there for linear regression has only one global, and no other local optima; thus gradient descent always converges (assuming the learning rate $\alpha$ is not too large) to the global minimum. Indeed, $J$ is a convex quadratic function. 

​	 There is an alternative to batch gradient descent that also works very well. Consider the following algorithm:
+ **Stochastic gradient descent**
---
![[Pasted image 20240625105332.png]]

​	即：
$$
\theta:=\theta +\alpha (y^{(i)}-h_{\theta}(x^{(i)}))\cdot x^{(i)}
$$
​	In this algorithm, we repeatedly run through the training set, and each time we encounter a training example, we update the parameters according to the gradient of the error with respect to that single training example only. This algorithm is called **stochastic gradient descent** (also **incremental gradient descent**). 

- **BGD vs SGD**
---
Whereas batch gradient descent has to scan through the entire training set before taking a single step—a costly operation if $n$ is large—stochastic gradient descent can start making progress right away, and continues to make progress with each example it looks at. Often, stochastic gradient descent gets $\theta$ "close" to the minimum much faster than batch gradient descent. (**Note however that it may never "converge" to the minimum, and the parameters $\theta$ will keep oscillating around the minimum of $J(\theta)$; but in practice most of the values near the minimum will be reasonably good approximations to the true minimum.** Also, by slowly letting the learning rate $\alpha$ decrease to zero as the algorithm runs, it is also possible to ensure that the parameters will converge to the global minimum rather merely oscillate around the minimum.) For these reasons, particularly when the training set is large, stochastic gradient descent is often preferred over batch gradient descent.

## 2 The normal equations 法方程

Another way different from gradient descent minimizes $J$. This time performing the minimization explicitly and without resorting to an iterative algorithm. In this method, we will minimize $J$ by explicitly taking its derivatives with respect to the $\theta_j$'s, and setting them to zero. Some notation for doing calculus with matrices.

### 2.1 Matrix derivatives

For a function $f:\mathbb{R}^{n\times d} \rightarrow \mathbb{R}$ mapping from $n$-by-$d$ matrices to the real numbers, we define the derivative of $f$ with respect to $A$ to be:
$$
\begin{array}{c}
\nabla_A f(A) = \left[
\begin{array}{ccc}
\frac{\partial f}{\partial A_{11}} & \cdots & \frac{\partial f}{\partial A_{1 d}}\\
\vdots & \ddots & \vdots\\
\frac{\partial f}{\partial A_{n 1}} & \cdots & \frac{\partial f}{\partial A_{n d }}
\end{array}
\right]
\end{array}
$$
Thus, the gradient $\nabla_A f(A)$ is itself an $n$-by-$d$ matrix, whose $(i,j)$-element is $\partial f /\partial A_{ij}$. For example, suppose $A=\left[ \begin{array}{c}A_{11}&A_{12}\\A_{21}&A_{22} \end{array}\right]$ is a 2-by-2 matrix, and the function $f:\mathbb{R}^{2\times 2}\rightarrow \mathbb{R}$ is given by
$$
f(A)=\frac{3}{2}A_{11}+5A_{12}^2+A_{21}A_{22}
$$
Here, $A_{ij}$ denotes the $(i,j)$ entry of  the matrix $A$. We then have
$$
\begin{array}{c}
\nabla_A f(A) = \left[
\begin{array}{cc}
\frac{3}{2}  & 10A_{12}\\
A_{2 2} & A_{21}
\end{array}
\right]
\end{array}
$$

### 2.2 Least squares revisited

Let us now proceed to find in closed-form the value of $\theta$ that minimizes $J(\theta)$. 

Given a training set, define the **design matrix $X$** to be the $n$-by-$d$ matrix(actually $n$-by-$d+1$, if we include the intercept term)

that contains the training examples' input values in its rows:
$$
\begin{array}{c}
X = \left[\begin{array}{c}
-(x^{(1)})^T-\\
-(x^{(2)})^T-\\
\vdots\\
-(x^{(n)})^T-\\
\end{array}
\right]
\end{array}
$$
Also, let $\vec{y}$ be the $n$-dimensional vector containing all the target values from the training set:
$$
\vec{y} = \left[
\begin{array}{c}
y^{(1)}\\
y^{(2)}\\
\vdots\\
y^{(n)}\\
\end{array}
\right]
$$
Now, since $h_{\theta}(x^{(i)}) = (x^{(i)})^T\theta$, we can easily verify that 
$$
\begin{array}{align}
X\theta - \vec{y} &=\left[\begin{array}{c}
(x^{(1)})^T\theta\\
\vdots\\
(x^{(n)})^T\theta
\end{array}\right]- 
\left[\begin{array}{c}
y^{(1)}\\
\vdots\\
y^{(n)}
\end{array}
\right]
\\&=\left[\begin{array}{c}
h_{\theta}(x^{(1)})-y^{(1)}\\
\vdots\\
h_{\theta}(x^{(n)})-y^{(n)}
\end{array}\right]
\end{array}
$$
Thus, using the fact that for a vector $z$, we have that $z^Tz=\sum_iz_i^2$:
$$
\begin{align}
\frac{1}{2}(X\theta-\vec{y})^T(X\theta -\vec{y}) &=\frac{1}{2} \sum_{i=1}^n(h_{\theta}(x^{(i)})-y^{(i)})^2\\
&=J(\theta)
\end{align}
$$
Hence,导数等于$0$的地方就是$J$函数的极值点，而该极值点就是全局最小值点.

$$
\begin{align}
\nabla_\theta J(\theta) &= \nabla_\theta \frac{1}{2} (X\theta - \vec{y})^T (X\theta - \vec{y}) \\
&= \frac{1}{2} \nabla_\theta \left( (X\theta)^T X\theta - (X\theta)^T \vec{y} - \vec{y}^T X\theta + \vec{y}^T \vec{y} \right) \\
&= \frac{1}{2} \nabla_\theta \left( \theta^T X^T X \theta - \vec{y}^T X \theta - \theta^T X^T \vec{y} + \vec{y}^T \vec{y} \right) \\
&= \frac{1}{2} \nabla_\theta \left( \theta^T X^T X \theta - 2 \vec{y}^T X \theta + \vec{y}^T \vec{y} \right) \\
&= \frac{1}{2} \left( 2 X^T X \theta - 2 X^T \vec{y} \right) \\
&= X^T X \theta - X^T \vec{y}
\end{align}
$$
To minimize $J$, we used the fact that $a^Tb=b^Ta$, and the facts $\nabla_x b^T x = b$ and $\nabla_x x^T A x = 2Ax$ for symmetric matrix $A$. We set its derivatives to zero, and obtain the **normal equations:**
$$
X^TX\theta = X^T\vec{y}
$$
Thus, the value of $\theta$ that minimizes $J(\theta)$ is given in closed form by the equation
$$
\theta = (X^TX)^{-1}X^T\vec{y}
$$
## 3 Probabilistic interpretation
