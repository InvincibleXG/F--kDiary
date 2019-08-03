# 基于Matrix-Toolkits-Java的多元线性回归系数拟合方法
---
最近要搞数据分析，但是**Py太慢了**，本身用SPSS跑线性回归都表示亚历山大，那么放到服务器上，自然要讲究性能。于是完全通过自己的目的性，在网上搜寻各种 **Java科学计算库** ，刚开始找到个jama，后来用的时候发现连Matrix类都么的，你分析你🐎呢？于是继续找，我不断地寻找，终于找到了一个 **以com.google冠名的Matrix-Toolkits-Java** ~

基于这个矩阵计算的高性能库，我编写了一套多元线性回归拟合误差最小的系数矩阵的方法，在此献丑。

---

```java
import java.text.DecimalFormat;

import no.uib.cipr.matrix.DenseMatrix;
import no.uib.cipr.matrix.Matrices;
import no.uib.cipr.matrix.Matrix;

/**
 * @author XG_WYF
 *  最小二乘法解多元线性回归 系数矩阵
 */
public class LinearRegression
{

    public static synchronized ResultObject leastSquareAlgorithm(Matrix x, Matrix y)
    {
        Matrix xT=x.transpose(new DenseMatrix(x.numColumns(), x.numRows()));
        Matrix xTx= multiply(xT, x);
        Matrix xTx_Reverse=invert((DenseMatrix) xTx);
        Matrix coefficient=multiply(multiply(xTx_Reverse, xT), y);
        ResultObject res=new LinearRegression.ResultObject();
        res.coefficient=coefficient;
        Matrix y2=multiply(x, coefficient);
        res.deviation=calculateDeviation(y, y2);
        return res;
    }

    /**
     *  矩阵 A B 相点乘
     * @param A m*k矩阵
     * @param B k*n 矩阵
     * @return 如果不可乘 返回null; 否则返回 m*n 矩阵
     */
    public static Matrix multiply(Matrix A, Matrix B)
    {
        if (A==null || B==null || A.numColumns()!=B.numRows()) return null;
        return A.mult(B, new DenseMatrix(A.numRows(), B.numColumns()));
    }

    /**
     *  对密集矩阵 A 求逆矩阵
     * @param A 可逆的密集矩阵
     * @return 若 A 不存在 返回null; 否则返回 A 的逆矩阵
     */
    public static Matrix invert(DenseMatrix A)
    {
        if (A==null) return null;
        DenseMatrix I = Matrices.identity(A.numRows());
        DenseMatrix AI = I.copy();
        return A.solve(I, AI);
    }

    /**
     *  计算线性函数值矩阵的偏差 多为一维列矩阵
     * @param y1 预测值矩阵或实际值矩阵
     * @param y2 实际值矩阵或预测值矩阵
     * @return 误差
     */
    public static double calculateDeviation(Matrix y1, Matrix y2)
    {
        if (y1.numRows()!=y2.numRows() || y1.numColumns()!=y2.numColumns()) return -1;
        Matrix delta=y1.add(-1, y2);
        double deviation=0;
        for (int i=0, l=delta.numColumns(); i<l; i++){
            double sum=0;
            for (int j=0, h=delta.numRows(); j<h; j++){
                sum+=Math.pow(delta.get(j, i), 2);
            }
            deviation+=Math.sqrt(sum);
        }
        return deviation/delta.numColumns();
    }

    public static class ResultObject
    {
        /**
         *  系数矩阵
         */
        Matrix coefficient;
        /**
         *  误差
         */
        Double deviation;

        public Matrix getCoefficient()
        {
            return coefficient;
        }

        protected void setCoefficient(Matrix coefficient)
        {
            this.coefficient = coefficient;
        }

        public Double getDeviation()
        {
            return deviation;
        }

        protected void setDeviation(Double deviation)
        {
            this.deviation = deviation;
        }

        @Override
        public String toString()
        {
            StringBuilder sb=new StringBuilder("系数矩阵为: \n");
            int rowLen=coefficient.numRows();
            int colLen=coefficient.numColumns();
            DecimalFormat df=new DecimalFormat("0.0000");
            for (int i=0; i<rowLen; i++){
                sb.append("| ");
                for (int j=0; j<colLen; j++){
                    String numbers=df.format(coefficient.get(i, j));
                    sb.append(numbers).append(" | ");
                }
                sb.append("\r\n");
            }
            sb.append("误差为: ").append(deviation);
            return sb.toString();
        }
    }
}
```

使用的MTJ依赖为

```xml
<dependency>
	<groupId>com.googlecode.matrix-toolkits-java</groupId>
	<artifactId>mtj</artifactId>
	<version>LATEST</version>
</dependency>
```

测试方法
无解线性方程组

  
	x1 + 3x2 -7x3 = -7.5
	2x1 + 5x2 + 4x3 = 5.2
	-3x1 - 7x2 - 2x3 = -7.5
	x1 + 4x2 - 12x3 = -15
	

```java
 public static void main(String[] args)
    {
        Matrix x=new DenseMatrix(new double[][]{
                {1, 3, -7},
                {2, 5, 4},
                {-3, -7, -2},
                {1, 4, -12}
        });
        Matrix y=new DenseMatrix(new double[][]{
                {-7.5},
                {5.2},
                {-7.5},
                {-15}
        });
        System.out.println(LinearRegression.leastSquareAlgorithm(x, y).toString());
    }
```
输出

	系数矩阵为: 
	| 12.0121 | 
	| -4.3593 | 
	| 0.8253 | 
	误差为: 0.8693182879212229