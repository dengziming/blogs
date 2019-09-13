---
date: 2018-11-24
title: "headfirst 设计模式"
author: "邓子明"
tags:
    - java
    - oop
categories:
    - 代码整洁之道
comment: true
---


设计模式很抽象，其实以前我也尝试学过，只不过只学了一些表面，没有吃透，所以再学一下。能把抽象讲具体的人，都很厉害。

这个网址有资料： https://refactoring.guru/design-patterns

## 策略模式

封装算法

## 观察者模式

java 自带了库

## 装饰模式

装饰器和原类实现同一个接口

## 工厂

Objectville

```java

// 第一个版本
Pizza orderPizza(){
	Pizza pizza = new Pizza();
	pizza.prepare();
	pizza.bake();
	return pizza;
}

// 第二个版本

Pizza orderPizza(String type){

	Pizza pizza ;
	if (type = "cheese"){
		pizza = new CheesePizza();
	}else if(){
	    pizza= new ....
	}
	
	pizza.prepare();
	pizza.bake();
	return pizza;
}

// version 3

class PizzaStore {

	SimplePizzaFactory factory;
	// constructor
	
	Pizza orderPizza(String type){
	
		Pizza pizza = factory.createPizza(type)
		
		pizza.prepare();
		pizza.bake();
		return pizza;
	}

}

class SimplePizzaFactory {

	Pizza createPizza(String type){
		Pizza pizza ;
		if (type = "cheese"){
			pizza = new CheesePizza();
		}else if(){
		    pizza= new ....
		}
		return pizza;
	}
}


// version 4 

class PizzaStore {
	SimplePizzaFactory factory;
	// constructor
	
	Pizza orderPizza(String type){
	
		Pizza pizza = factory.createPizza(type)
		
		pizza.prepare();
		pizza.bake();
		return pizza;
	}
}

abstrat class PizzaFactory {
    Pizza createPizza(String type);
}

class NYPizzaFactory extends PizzaFactory{
	Pizza createPizza(String type){
	    Pizza pizza ;
		if (type = "cheese"){
			pizza = new NYCheesePizza();
		}else if(){
		    pizza= new ....
		}
		return pizza;
	}
}

class ChicagoPizzaFactory extends PizzaFactory{

	Pizza createPizza(String type){
	    Pizza pizza ;
		if (type = "cheese"){
			pizza = new ChicagoCheesePizza();
		}else if(){
		    pizza= new ....
		}
		return pizza;
	}
}

// version 5

abstract class PizzaStore {
	
	
	Pizza orderPizza(String type){
	
		Pizza pizza = createPizza(type)
		
		pizza.prepare();
		pizza.bake();
		return pizza;
	}
	
	abstract Pizza createPizza(type);
}

class NYPizzaStore extends PizzaStore{
	Pizza createPizza(type){
	    if (type = "cheese"){
			pizza = new NYCheesePizza();
		}else if(){
		    pizza= new ....
		}
	}
}

class ChicagoPizzaStore extends PizzaStore{
	Pizza createPizza(type){
		if (type = "cheese"){
			pizza = new ChicagoCheesePizza();
		}else if(){
		    pizza= new ....
		}
	}
}

// version 6

class ComPizzaStore{
	Pizza createPizza(string style, String type){
	    if (style = "NY"){
	        if (type = "cheese"){
	            // 
	        }
	    }
	    if (style = "NY"){
	    	if (type = ""){
	    	    //
	    	}
	    }
	}
}

// version 7  add ingredients

interface PizzaIngredientFactory {
	createSoil();
	createClam();
}

class NYPizzaIngredientFactory extends PizzaIngredientFactory{}
...
class ChicagoPizzaIngredientFactory extends PizzaIngredientFactory{}
...

abstract class Pizza {
    string soil;
    string clam;
    
    void prepare();
    void bake();
}

class CheesePizza extends Pizza{
    PizzaIngredientFactory ingerdientFactory;
    
    // constructor
    
    void prepare(){
    	soil = ingerdientFactory.createSoil();
    	clam = ingerdientFactory.createClam();
    }
}


```

## 单例

这个熟悉了

## 命令模式

