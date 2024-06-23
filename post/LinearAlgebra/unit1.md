# Unit I: Ax = b and the Four Subspaces



![img](./images/50ce4d8cddfa06b9c4d84f7e03a7e0e7_Unit_1_WIDE.jpg)

Four Fundamental Subspaces. 

## The Geometry of Linear Equations 

Three ways of thinking about systems of linear equations.

+ row method
+ column method
+ matrix method

$$
\begin{align*}
 2x - y = 0 \\
 -x+2y = 3
\end{align*}
$$

### Row Picture

一行看作是一条直线，两条方程代表两条直线，两条直线相交的点即为方程组解。

### Column Picture

在列视角中，重写方程组
$$
x
\begin{bmatrix}
	\begin{array}{r}
		2 \\
		-1  
	\end{array}
\end{bmatrix}
+y
\begin{bmatrix}
\begin{array}{r}
-1\\
2
\end{array}
\end{bmatrix}
=
\begin{bmatrix}
0\\
3
\end{bmatrix}
$$
Given two vectors $\mathbf{c}$ and $\mathbf{d}$ and scalars $x$ and $y$, the sum $x\mathbf{c} + y\mathbf{d}$ is called a *linear combination* of $\mathbf{c}$ and $\mathbf{d}$.  

看成列向量的线性组合。

<div style="text-align:center;">
    <img src="./images/image-20240623152755175.png" alt="Image Description">
</div>

### Matrix Picture

重写方程组
$$
\begin{bmatrix}
\begin{array}{rr}
2 &&-1\\
-1&&2
\end{array}
\end{bmatrix}
\begin{bmatrix}
x\\y
\end{bmatrix}=
\begin{bmatrix}
0\\3
\end{bmatrix}
$$
即
$$
A\mathbf{x}=\mathbf{b}
$$

### Linear Independence

In the column and matrix pictures, the right hand side of the equation is a vector $\mathbf{b}$. Given a matrix $A$, can we solve:
$$
A\mathbf{x}=\mathbf{b}
$$
for every possible vector $\mathbf{b}$? In other words, do the linear combinations of the column vectors fill the $xy$-plane (or space, in the three dimensional case)?

​	If the answer is "no", we say that $A$ is a *singular matrix*（奇异矩阵). In this singular case its column vectors are *linearly dependent*(线性相关);矩阵称为不可逆矩阵(*not invertible*) all linear combinations of those vectors lie on a point or line (in two dimensions) or on a point, line or plane (in three dimensions). The combinations don't fill the whole space.

## An overview of key ideas

​	Linear algebra progresses from vectors to matrices to subspaces.

### Subspaces

​	The ***basis*** of a vector space is a set of linearly independent vectors that **span** the full space.	

​	A *basis* for $\mathbb{R}^n$ is a collection of $n$ independent vectors in $\mathbb{R}^n$. Equivalently, a basis is a collection of $n$ vectors whose combinations cover the whole space. Or, a collection of vectors forms a basis whenever a matrix which has those vectors as its columns is invertible.

​	A *vector space* is a collection of vectors that is closed under linear combinations. A *subspace* is a vector space inside another vector space; a plane through the origin in $\mathbb{R}^3$ is an example of a subspace. A subspace  could be the equal to the space it's contained in; the smallest subspace contains only the zero vector. 

​	The subspaces of $\mathbb{R}^3$ are:

+ the origin,
+ a line through the origin,
+ a plane through the origin,
+ all of $\mathbb{R}^3$



