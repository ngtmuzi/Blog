---
title: javaScript实现一个分数计算的类
desc: 难得写点有趣的东西
author: ngtmuzi
category: 神秘代码
date: 2018-05-25 15:48:17
tags: 
- javascript
- Class
---

一开始实现的时候钻到了牛角尖里，还想着要保留复杂分式的所有结构，那就要构建成树还要做递归遍历，各种计算也复杂得要死，后来想了一下每次计算都直接化成最简式不就好了

```javascript

/**
 * @property {Number} numerator  分子
 * @property {Number} denominator  分母
 */
class Fraction {
  /**
   * 可以接受数值和分式
   * @param {Number|Fraction} numerator
   * @param {Number|Fraction} denominator
   * @return {Fraction}
   */
  constructor(numerator, denominator = 1) {
    if (Number.isInteger(numerator) && Number.isInteger(denominator)) {
      this.numerator   = numerator;
      this.denominator = denominator;
    } else if (numerator instanceof Fraction || denominator instanceof Fraction) {
      numerator   = numerator instanceof Fraction ? numerator : new Fraction(numerator, 1);
      denominator = denominator instanceof Fraction ? denominator : new Fraction(denominator, 1);

      return numerator.clone().divide(denominator);
    } else {
      throw new TypeError('prams must be Number or Fraction');
    }
    this.simplify();
  }

  /**
   * 辗转相除法求最大公约数
   * @param num1
   * @param num2
   * @return {number}
   */
  static greatestCommonDivisor(num1, num2) {
    let lesser  = Math.abs(num1);
    let greater = Math.abs(num2);

    while (lesser !== 0) {
      let t   = lesser;
      lesser  = greater % lesser;
      greater = t;
    }
    return greater;
  }

  clone() {
    return new Fraction(this.numerator, this.denominator);
  }

  /**
   * 化简分式
   * @return {Fraction}
   */
  simplify() {
    const gcd        = Fraction.greatestCommonDivisor(this.numerator, this.denominator);
    this.numerator   = this.numerator / gcd;
    this.denominator = this.denominator / gcd;
    return this;
  }

  /**
   * 导出格式  a/b
   * @return {string}
   */
  toString() {
    return `${this.numerator}/${this.denominator}`;
  }

  /**
   * 从字符串解析出分式，支持 1/3 和  3 的格式
   * @param str
   * @return {Fraction}
   */
  static fromString(str = '') {
    let arr = str.split('/').map(Number).filter(Number.isInteger);
    if (arr.length === 1) arr.push(1);
    if (arr.length !== 2) throw TypeError('params must be 2 Integer spread by "/"');
    return new Fraction(arr[0], arr[1]);
  }

  get value() {
    return this.numerator / this.denominator;
  }

  static add(frac1, frac2) {
    frac1 = frac1 instanceof Fraction ? frac1.clone() : new Fraction(frac1);
    frac2 = frac2 instanceof Fraction ? frac2.clone() : new Fraction(frac2);

    //  a/b + c/d = (a*d+b*c) / b*d
    frac1.numerator   = frac1.numerator * frac2.denominator + frac1.denominator * frac2.numerator;
    frac1.denominator = frac1.denominator * frac2.denominator;
    return frac1.simplify();
  }

  add(frac) {
    return Fraction.add(this, frac);
  }

  static subtract(frac1, frac2) {
    frac1 = frac1 instanceof Fraction ? frac1.clone() : new Fraction(frac1);
    frac2 = frac2 instanceof Fraction ? frac2.clone() : new Fraction(frac2);

    //  a/b + c/d = (a*d-b*c) / b*d
    frac1.numerator   = frac1.numerator * frac2.denominator - frac1.denominator * frac2.numerator;
    frac1.denominator = frac1.denominator * frac2.denominator;
    return frac1.simplify();
  }

  subtract(frac) {
    return Fraction.subtract(this, frac);
  }

  static multiply(frac1, frac2) {
    frac1 = frac1 instanceof Fraction ? frac1.clone() : new Fraction(frac1);
    frac2 = frac2 instanceof Fraction ? frac2.clone() : new Fraction(frac2);

    // (a/b)*(c/d) = (a*b)/(c*d)
    frac1.numerator   = frac1.numerator * frac2.numerator;
    frac1.denominator = frac1.denominator * frac2.denominator;
    return frac1.simplify();
  }

  multiply(frac) {
    return Fraction.multiply(this, frac);
  }

  static divide(frac1, frac2) {
    frac1 = frac1 instanceof Fraction ? frac1.clone() : new Fraction(frac1);
    frac2 = frac2 instanceof Fraction ? frac2.clone() : new Fraction(frac2);

    //  (a/b) / (c/d) = (a*d) / (b*c)
    frac1.numerator   = frac1.numerator * frac2.denominator;
    frac1.denominator = frac1.denominator * frac2.numerator;
    return frac1.simplify();
  }

  divide(frac) {
    return Fraction.divide(this, frac);
  }

}
```
业务代码写得多了居然连这种简单的功能代码都生疏了，老了老了，原来计划的每月一篇博客肯定是鸽了