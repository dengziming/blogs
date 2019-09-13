---
date: 2018-03-22
title: "guava学习"
author: "邓子明"
tags:
    - guava
categories:
    - java
comment: true
---

# 

https://www.tutorialspoint.com/guava/guava_optional_class.htm

## Optional

```java
import com.google.common.base.Optional;

public class GuavaTester {
   public static void main(String args[]) {
      GuavaTester guavaTester = new GuavaTester();

      Integer value1 =  null;
      Integer value2 =  new Integer(10);
      
      //Optional.fromNullable - allows passed parameter to be null.
      Optional<Integer> a = Optional.fromNullable(value1);
      
      //Optional.of - throws NullPointerException if passed parameter is null
      Optional<Integer> b = Optional.of(value2);		

      System.out.println(guavaTester.sum(a,b));
   }

   public Integer sum(Optional<Integer> a, Optional<Integer> b) {
      //Optional.isPresent - checks the value is present or not
      System.out.println("First parameter is present: " + a.isPresent());

      System.out.println("Second parameter is present: " + b.isPresent());

      //Optional.or - returns the value if present otherwise returns
      //the default value passed.
      Integer value1 = a.or(new Integer(0));	

      //Optional.get - gets the value, value should be present
      Integer value2 = b.get();

      return value1 + value2;
   }	
}
```
Optional 主要方法为 fromNullable 、of 、isPresent 、or 、 get
结果：
```
First parameter is present: false
Second parameter is present: true
10
```

## Preconditions

Preconditions 用来检查参数

```java
import com.google.common.base.Preconditions;

public class GuavaTester {

   public static void main(String args[]) {
      GuavaTester guavaTester = new GuavaTester();

      try {
         System.out.println(guavaTester.sqrt(-3.0));
      } catch(IllegalArgumentException e) {
         System.out.println(e.getMessage());
      }

      try {
         System.out.println(guavaTester.sum(null,3));
      } catch(NullPointerException e) {
         System.out.println(e.getMessage());
      }

      try {
         System.out.println(guavaTester.getValue(6));
      } catch(IndexOutOfBoundsException e) {
         System.out.println(e.getMessage());
      }
   }

   public double sqrt(double input) throws IllegalArgumentException {
      Preconditions.checkArgument(input > 0.0,
         "Illegal Argument passed: Negative value %s.", input);
      return Math.sqrt(input);
   }

   public int sum(Integer a, Integer b) {
      a = Preconditions.checkNotNull(a, "Illegal Argument passed: First parameter is Null.");
      b = Preconditions.checkNotNull(b, "Illegal Argument passed: Second parameter is Null.");

      return a+b;
   }

   public int getValue(int input) {
      int[] data = {1,2,3,4,5};
      Preconditions.checkElementIndex(input,data.length, "Illegal Argument passed: Invalid index.");
      return 0;
   }
}
```

可以看出主要方法： checkArgument 、 checkNotNull 、checkElementIndex

返回结果：
```java
Illegal Argument passed: Negative value -3.0.
Illegal Argument passed: First parameter is Null.
Illegal Argument passed: Invalid index. (6) must be less than size (5)
```

## Ordering

enriched comparator

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import com.google.common.collect.Ordering;

public class GuavaTester {
   public static void main(String args[]) {
      List<Integer> numbers = new ArrayList<Integer>();
      
      numbers.add(new Integer(5));
      numbers.add(new Integer(2));
      numbers.add(new Integer(15));
      numbers.add(new Integer(51));
      numbers.add(new Integer(53));
      numbers.add(new Integer(35));
      numbers.add(new Integer(45));
      numbers.add(new Integer(32));
      numbers.add(new Integer(43));
      numbers.add(new Integer(16));

      Ordering ordering = Ordering.natural();
      System.out.println("Input List: ");
      System.out.println(numbers);		
         
      Collections.sort(numbers,ordering );
      System.out.println("Sorted List: ");
      System.out.println(numbers);
         
      System.out.println("======================");
      System.out.println("List is sorted: " + ordering.isOrdered(numbers));
      System.out.println("Minimum: " + ordering.min(numbers));
      System.out.println("Maximum: " + ordering.max(numbers));
         
      Collections.sort(numbers,ordering.reverse());
      System.out.println("Reverse: " + numbers);

      numbers.add(null);
      System.out.println("Null added to Sorted List: ");
      System.out.println(numbers);		

      Collections.sort(numbers,ordering.nullsFirst());
      System.out.println("Null first Sorted List: ");
      System.out.println(numbers);
      System.out.println("======================");

      List<String> names = new ArrayList<String>();
      
      names.add("Ram");
      names.add("Shyam");
      names.add("Mohan");
      names.add("Sohan");
      names.add("Ramesh");
      names.add("Suresh");
      names.add("Naresh");
      names.add("Mahesh");
      names.add(null);
      names.add("Vikas");
      names.add("Deepak");

      System.out.println("Another List: ");
      System.out.println(names);

      Collections.sort(names,ordering.nullsFirst().reverse());
      System.out.println("Null first then reverse sorted list: ");
      System.out.println(names);
   }
}
```

可以看出主要方法：natural、Collections.sort(numbers,ordering ) 、min、max、reverse、nullsFirst

结果：
```java
Input List: 
[5, 2, 15, 51, 53, 35, 45, 32, 43, 16]
Sorted List: 
[2, 5, 15, 16, 32, 35, 43, 45, 51, 53]
======================
List is sorted: true
Minimum: 2
Maximum: 53
Reverse: [53, 51, 45, 43, 35, 32, 16, 15, 5, 2]
Null added to Sorted List: 
[53, 51, 45, 43, 35, 32, 16, 15, 5, 2, null]
Null first Sorted List: 
[null, 2, 5, 15, 16, 32, 35, 43, 45, 51, 53]
======================
Another List: 
[Ram, Shyam, Mohan, Sohan, Ramesh, Suresh, Naresh, Mahesh, null, Vikas, Deepak]
Null first then reverse sorted list: 
[Vikas, Suresh, Sohan, Shyam, Ramesh, Ram, Naresh, Mohan, Mahesh, Deepak, null]
```

## Objects

Objects class provides helper functions applicable to all objects such as equals, hashCode, etc.

```java
import com.google.common.base.Objects;

public class GuavaTester {
   public static void main(String args[]) {
      Student s1 = new Student("Mahesh", "Parashar", 1, "VI");	
      Student s2 = new Student("Suresh", null, 3, null);	
	  
      System.out.println(s1.equals(s2));
      System.out.println(s1.hashCode());	
      System.out.println(
         Objects.toStringHelper(s1)
         .add("Name",s1.getFirstName()+" " + s1.getLastName())
         .add("Class", s1.getClassName())
         .add("Roll No", s1.getRollNo())
         .toString());
   }
}

class Student {
   private String firstName;
   private String lastName;
   private int rollNo;
   private String className;

   public Student(String firstName, String lastName, int rollNo, String className) {
      this.firstName = firstName;
      this.lastName = lastName;
      this.rollNo = rollNo;
      this.className = className;		
   }

   @Override
   public boolean equals(Object object) {
      if(!(object instanceof Student) || object == null) {
         return false;
      }
      Student student = (Student)object;
      // no need to handle null here		
      // Objects.equal("test", "test") == true
      // Objects.equal("test", null) == false
      // Objects.equal(null, "test") == false
      // Objects.equal(null, null) == true		
      return Objects.equal(firstName, student.firstName)  // first name can be null
         && Objects.equal(lastName, student.lastName)     // last name can be null
         && Objects.equal(rollNo, student.rollNo)	
         && Objects.equal(className, student.className);  // class name can be null
   }

   @Override
   public int hashCode() {
      //no need to compute hashCode by self
      return Objects.hashCode(className,rollNo);
   }
}
```

可以看出主要方法： hashCode、equal、toStringHelper 等

结果
```false
85871
Student{Name=Mahesh Parashar, Class=VI, Roll No=1}
```

## Range

类似 scala

```java
import com.google.common.collect.ContiguousSet;
import com.google.common.collect.DiscreteDomain;
import com.google.common.collect.Range;
import com.google.common.primitives.Ints;

public class GuavaTester {

   public static void main(String args[]) {
      GuavaTester tester = new GuavaTester();
      tester.testRange();
   }

   private void testRange() {

      //create a range [a,b] = { x | a <= x <= b}
      Range<Integer> range1 = Range.closed(0, 9);	
      System.out.print("[0,9] : ");
      printRange(range1);		
      
      System.out.println("5 is present: " + range1.contains(5));
      System.out.println("(1,2,3) is present: " + range1.containsAll(Ints.asList(1, 2, 3)));
      System.out.println("Lower Bound: " + range1.lowerEndpoint());
      System.out.println("Upper Bound: " + range1.upperEndpoint());

      //create a range (a,b) = { x | a < x < b}
      Range<Integer> range2 = Range.open(0, 9);
      System.out.print("(0,9) : ");
      printRange(range2);

      //create a range (a,b] = { x | a < x <= b}
      Range<Integer> range3 = Range.openClosed(0, 9);
      System.out.print("(0,9] : ");
      printRange(range3);

      //create a range [a,b) = { x | a <= x < b}
      Range<Integer> range4 = Range.closedOpen(0, 9);
      System.out.print("[0,9) : ");
      printRange(range4);

      //create an open ended range (9, infinity
      Range<Integer> range5 = Range.greaterThan(9);
      System.out.println("(9,infinity) : ");
      System.out.println("Lower Bound: " + range5.lowerEndpoint());
      System.out.println("Upper Bound present: " + range5.hasUpperBound());

      Range<Integer> range6 = Range.closed(3, 5);	
      printRange(range6);

      //check a subrange [3,5] in [0,9]
      System.out.println("[0,9] encloses [3,5]:" + range1.encloses(range6));

      Range<Integer> range7 = Range.closed(9, 20);	
      printRange(range7);
      
      //check ranges to be connected		
      System.out.println("[0,9] is connected [9,20]:" + range1.isConnected(range7));
      Range<Integer> range8 = Range.closed(5, 15);	

      //intersection
      printRange(range1.intersection(range8));

      //span
      printRange(range1.span(range8));
   }

   private void printRange(Range<Integer> range) {		
   
      System.out.print("[ ");
      
      for(int grade : ContiguousSet.create(range, DiscreteDomain.integers())) {
         System.out.print(grade +" ");
      }
      System.out.println("]");
   }
}
```

主要方法：contains 、containsAll、lowerEndpoint、upperEndpoint、open、closed、closedOpen、openClosed、greaterThan、hasUpperBound、encloses

结果：
```
[0,9] : [ 0 1 2 3 4 5 6 7 8 9 ]
5 is present: true
(1,2,3) is present: true
Lower Bound: 0
Upper Bound: 9
(0,9) : [ 1 2 3 4 5 6 7 8 ]
(0,9] : [ 1 2 3 4 5 6 7 8 9 ]
[0,9) : [ 0 1 2 3 4 5 6 7 8 ]
(9,infinity) : 
Lower Bound: 9
Upper Bound present: false
[ 3 4 5 ]
[0,9] encloses [3,5]:true
[ 9 10 11 12 13 14 15 16 17 18 19 20 ]
[0,9] is connected [9,20]:true
[ 5 6 7 8 9 ]
[ 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 ]
```

## Throwable

抛出异常：
```java
import java.io.IOException;

import com.google.common.base.Objects;
import com.google.common.base.Throwables;

public class GuavaTester {
   public static void main(String args[]) {
   
      GuavaTester tester = new GuavaTester();

      try {
         tester.showcaseThrowables();
         
      } catch (InvalidInputException e) {
         //get the root cause
         System.out.println(Throwables.getRootCause(e));
      
      } catch (Exception e) {
         //get the stack trace in string format
         System.out.println(Throwables.getStackTraceAsString(e));
      }

      try {
         tester.showcaseThrowables1();

      } catch (Exception e) {
         System.out.println(Throwables.getStackTraceAsString(e));
      }
   }

   public void showcaseThrowables() throws InvalidInputException {
      try {
         sqrt(-3.0);
      } catch (Throwable e) {
         //check the type of exception and throw it
         Throwables.propagateIfInstanceOf(e, InvalidInputException.class);
         Throwables.propagate(e);
      }
   }

   public void showcaseThrowables1() {
      try {
         int[] data = {1,2,3};
         getValue(data, 4);
      } catch (Throwable e) {
         Throwables.propagateIfInstanceOf(e, IndexOutOfBoundsException.class);
         Throwables.propagate(e);
      }
   }

   public double sqrt(double input) throws InvalidInputException {
      if(input < 0) throw new InvalidInputException();
      return Math.sqrt(input);
   }

   public double getValue(int[] list, int index) throws IndexOutOfBoundsException {
      return list[index];
   }

   public void dummyIO() throws IOException {
      throw new IOException();
   }
}

class InvalidInputException extends Exception {
}
```

主要方法：
getStackTraceAsString、propagate、propagateIfInstanceOf、getRootCause
结果：
```
InvalidInputException
java.lang.ArrayIndexOutOfBoundsException: 4
   at GuavaTester.getValue(GuavaTester.java:52)
   at GuavaTester.showcaseThrowables1(GuavaTester.java:38)
   at GuavaTester.main(GuavaTester.java:19)
```

## Collections

### Multiset

可以有重复元素

```java
import java.util.Iterator;
import java.util.Set;

import com.google.common.collect.HashMultiset;
import com.google.common.collect.Multiset;

public class GuavaTester {

   public static void main(String args[]) {
   
      //create a multiset collection
      Multiset<String> multiset = HashMultiset.create();
      
      multiset.add("a");
      multiset.add("b");
      multiset.add("c");
      multiset.add("d");
      multiset.add("a");
      multiset.add("b");
      multiset.add("c");
      multiset.add("b");
      multiset.add("b");
      multiset.add("b");
      
      //print the occurrence of an element
      System.out.println("Occurrence of 'b' : "+multiset.count("b"));
      
      //print the total size of the multiset
      System.out.println("Total Size : "+multiset.size());
      
      //get the distinct elements of the multiset as set
      Set<String> set = multiset.elementSet();

      //display the elements of the set
      System.out.println("Set [");
      
      for (String s : set) {
         System.out.println(s);
      }

      System.out.println("]");
      
      //display all the elements of the multiset using iterator
      Iterator<String> iterator  = multiset.iterator();
      System.out.println("MultiSet [");

      while(iterator.hasNext()) {
         System.out.println(iterator.next());
      }
      
      System.out.println("]");
      
      //display the distinct elements of the multiset with their occurrence count
      System.out.println("MultiSet [");

      for (Multiset.Entry<String> entry : multiset.entrySet()) {
         System.out.println("Element: " + entry.getElement() + ", Occurrence(s): " + entry.getCount());
      }
      System.out.println("]");

      //remove extra occurrences
      multiset.remove("b",2);
      
      //print the occurrence of an element
      System.out.println("Occurence of 'b' : " + multiset.count("b"));
   }
}
```

主要方法：HashMultiset.create();、 add 、count、size、elementSet、iterator、entrySet

### Multimap
和MultiSet类似，每个 key 可以隐射多个值

```java
import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Set;

import com.google.common.collect.ArrayListMultimap;
import com.google.common.collect.Multimap;

public class GuavaTester {
   public static void main(String args[]) {
   
      GuavaTester tester = new GuavaTester();
      Multimap<String,String> multimap = tester.getMultimap();

      List<String> lowerList = (List<String>)multimap.get("lower");
      System.out.println("Initial lower case list");
      System.out.println(lowerList.toString());

      lowerList.add("f");
      System.out.println("Modified lower case list");
      System.out.println(lowerList.toString());

      List<String> upperList = (List<String>)multimap.get("upper");
      System.out.println("Initial upper case list");
      System.out.println(upperList.toString());

      upperList.remove("D");
      System.out.println("Modified upper case list");
      System.out.println(upperList.toString());

      Map<String, Collection<String>> map = multimap.asMap();
      System.out.println("Multimap as a map");

      for (Map.Entry<String,  Collection<String>> entry : map.entrySet()) {
         String key = entry.getKey();
         Collection<String> value =  multimap.get("lower");
         System.out.println(key + ":" + value);
      }

      System.out.println("Keys of Multimap");
      Set<String> keys =  multimap.keySet();

      for(String key:keys) {
         System.out.println(key);
      }

      System.out.println("Values of Multimap");
      Collection<String> values = multimap.values();
      System.out.println(values);
   }

   private Multimap<String,String> getMultimap() {

      //Map<String, List<String>>
      // lower -> a, b, c, d, e
      // upper -> A, B, C, D

      Multimap<String,String> multimap = ArrayListMultimap.create();

      multimap.put("lower", "a");
      multimap.put("lower", "b");
      multimap.put("lower", "c");
      multimap.put("lower", "d");
      multimap.put("lower", "e");

      multimap.put("upper", "A");
      multimap.put("upper", "B");
      multimap.put("upper", "C");
      multimap.put("upper", "D");		

      return multimap;
   }
}
```

结果：
```java
Initial lower case list
[a, b, c, d, e]
Modified lower case list
[a, b, c, d, e, f]
Initial upper case list
[A, B, C, D]
Modified upper case list
[A, B, C]
Multimap as a map
upper:[a, b, c, d, e, f]
lower:[a, b, c, d, e, f]
Keys of Multimap
upper
lower
Values of Multimap
[a, b, c, d, e, f, A, B, C]
```

### Bimap

A BiMap is a special kind of map which maintains an inverse view of the map while ensuring that no duplicate values are present in the map and a value can be used safely to get the key back.

biMap 内部将 key value进行反转，保存了一个 value-key的对，保证 Value不重复。

```java
import com.google.common.collect.BiMap;
import com.google.common.collect.HashBiMap;

public class GuavaTester {

   public static void main(String args[]) {
      BiMap<Integer, String> empIDNameMap = HashBiMap.create();

      empIDNameMap.put(new Integer(101), "Mahesh");
      empIDNameMap.put(new Integer(102), "Sohan");
      empIDNameMap.put(new Integer(103), "Ramesh");

      //Emp Id of Employee "Mahesh"
      System.out.println(empIDNameMap.inverse().get("Mahesh"));
   }	
}
```

结果：

```
101
```

### 

It is similar to creating a map of maps.

```java
import java.util.Map;
import java.util.Set;

import com.google.common.collect.HashBasedTable;
import com.google.common.collect.Table;

public class GuavaTester {
   public static void main(String args[]) {
   
      //Table<R,C,V> == Map<R,Map<C,V>>
      /*
      *  Company: IBM, Microsoft, TCS
      *  IBM 		-> {101:Mahesh, 102:Ramesh, 103:Suresh}
      *  Microsoft 	-> {101:Sohan, 102:Mohan, 103:Rohan } 
      *  TCS 		-> {101:Ram, 102: Shyam, 103: Sunil } 
      * 
      * */
      
      //create a table
      Table<String, String, String> employeeTable = HashBasedTable.create();

      //initialize the table with employee details
      employeeTable.put("IBM", "101","Mahesh");
      employeeTable.put("IBM", "102","Ramesh");
      employeeTable.put("IBM", "103","Suresh");

      employeeTable.put("Microsoft", "111","Sohan");
      employeeTable.put("Microsoft", "112","Mohan");
      employeeTable.put("Microsoft", "113","Rohan");

      employeeTable.put("TCS", "121","Ram");
      employeeTable.put("TCS", "122","Shyam");
      employeeTable.put("TCS", "123","Sunil");

      //get Map corresponding to IBM
      Map<String,String> ibmEmployees =  employeeTable.row("IBM");

      System.out.println("List of IBM Employees");
      
      for(Map.Entry<String, String> entry : ibmEmployees.entrySet()) {
         System.out.println("Emp Id: " + entry.getKey() + ", Name: " + entry.getValue());
      }

      //get all the unique keys of the table
      Set<String> employers = employeeTable.rowKeySet();
      System.out.print("Employers: ");
      
      for(String employer: employers) {
         System.out.print(employer + " ");
      }
      
      System.out.println();

      //get a Map corresponding to 102
      Map<String,String> EmployerMap =  employeeTable.column("102");
      
      for(Map.Entry<String, String> entry : EmployerMap.entrySet()) {
         System.out.println("Employer: " + entry.getKey() + ", Name: " + entry.getValue());
      }		
   }	
}
```

## LoadingCache

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

import com.google.common.base.MoreObjects;
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;

public class GuavaTester {
   public static void main(String args[]) {
   
      //create a cache for employees based on their employee id
      LoadingCache<String, Employee> employeeCache = 
         CacheBuilder.newBuilder()
         .maximumSize(100)                             // maximum 100 records can be cached
         .expireAfterAccess(30, TimeUnit.MINUTES)      // cache will expire after 30 minutes of access
         .build(new CacheLoader<String, Employee>() {  // build the cacheloader
            
            @Override
            public Employee load(String empId) throws Exception {
               //make the expensive call
               return getFromDatabase(empId);
            } 
         });

      try {			
         //on first invocation, cache will be populated with corresponding
         //employee record
         System.out.println("Invocation #1");
         System.out.println(employeeCache.get("100"));
         System.out.println(employeeCache.get("103"));
         System.out.println(employeeCache.get("110"));
         
         //second invocation, data will be returned from cache
         System.out.println("Invocation #2");
         System.out.println(employeeCache.get("100"));
         System.out.println(employeeCache.get("103"));
         System.out.println(employeeCache.get("110"));

      } catch (ExecutionException e) {
         e.printStackTrace();
      }
   }

   private static Employee getFromDatabase(String empId) {
   
      Employee e1 = new Employee("Mahesh", "Finance", "100");
      Employee e2 = new Employee("Rohan", "IT", "103");
      Employee e3 = new Employee("Sohan", "Admin", "110");

      Map<String, Employee> database = new HashMap<String, Employee>();
      
      database.put("100", e1);
      database.put("103", e2);
      database.put("110", e3);
      
      System.out.println("Database hit for" + empId);
      
      return database.get(empId);		
   }
}

class Employee {
   String name;
   String dept;
   String emplD;

   public Employee(String name, String dept, String empID) {
      this.name = name;
      this.dept = dept;
      this.emplD = empID;
   }
   
   public String getName() {
      return name;
   }
   
   public void setName(String name) {
      this.name = name;
   }
   
   public String getDept() {
      return dept;
   }
   
   public void setDept(String dept) {
      this.dept = dept;
   }
   
   public String getEmplD() {
      return emplD;
   }
   
   public void setEmplD(String emplD) {
      this.emplD = emplD;
   }

   @Override
   public String toString() {
      return MoreObjects.toStringHelper(Employee.class)
      .add("Name", name)
      .add("Department", dept)
      .add("Emp Id", emplD).toString();
   }	
}
```

结果

```java
Invocation #1
Database hit for100
Employee{Name=Mahesh, Department=Finance, Emp Id=100}
Database hit for103
Employee{Name=Rohan, Department=IT, Emp Id=103}
Database hit for110
Employee{Name=Sohan, Department=Admin, Emp Id=110}
Invocation #2
Employee{Name=Mahesh, Department=Finance, Emp Id=100}
Employee{Name=Rohan, Department=IT, Emp Id=103}
Employee{Name=Sohan, Department=Admin, Emp Id=110}
```

可以看出 第一次 invocation 去查数据，第二次直接读的缓存。


## String

### Joiner

```java
import java.util.Arrays;
import com.google.common.base.Joiner;

public class GuavaTester {
   public static void main(String args[]) {
      GuavaTester tester = new GuavaTester();
      tester.testJoiner();	
   }

   private void testJoiner() {
      System.out.println(Joiner.on(",")
         .skipNulls()
         .join(Arrays.asList(1,2,3,4,5,null,6)));
   }
}
```

结果：`1,2,3,4,5,6`

### Splitter

```java
import com.google.common.base.Splitter;

public class GuavaTester {
   public static void main(String args[]) {
      GuavaTester tester = new GuavaTester();
      tester.testSplitter();
   }

   private void testSplitter() {
      System.out.println(Splitter.on(',')
         .trimResults()
         .omitEmptyStrings()
         .split("the ,quick, ,brown, fox, jumps, over, the, lazy, little dog."));
   }
}
```

结果：[the, quick, brown, fox, jumps, over, the, lazy, little dog.]

### CharMatcher

```java
import com.google.common.base.CharMatcher;
import com.google.common.base.Splitter;

public class GuavaTester {
   public static void main(String args[]) {
      GuavaTester tester = new GuavaTester();
      tester.testCharMatcher();
   }

   private void testCharMatcher() {
      System.out.println(CharMatcher.DIGIT.retainFrom("mahesh123"));    // only the digits
      System.out.println(CharMatcher.WHITESPACE.trimAndCollapseFrom("     Mahesh     Parashar ", ' '));

      // trim whitespace at ends, and replace/collapse whitespace into single spaces
      System.out.println(CharMatcher.JAVA_DIGIT.replaceFrom("mahesh123", "*"));  // star out all digits
      System.out.println(CharMatcher.JAVA_DIGIT.or(CharMatcher.JAVA_LOWER_CASE).retainFrom("mahesh123"));

      // eliminate all characters that aren't digits or lowercase
   }
}
```

结果：
```
123
Mahesh Parashar
mahesh***
mahesh123
```

### CaseFormat

```java
import com.google.common.base.CaseFormat;

public class GuavaTester {
   public static void main(String args[]) {
      GuavaTester tester = new GuavaTester();
      tester.testCaseFormat();
   }

   private void testCaseFormat() {
      String data = "test_data";
      System.out.println(CaseFormat.LOWER_HYPHEN.to(CaseFormat.LOWER_CAMEL, "test-data"));
      System.out.println(CaseFormat.LOWER_UNDERSCORE.to(CaseFormat.LOWER_CAMEL, "test_data"));
      System.out.println(CaseFormat.UPPER_UNDERSCORE.to(CaseFormat.UPPER_CAMEL, "test_data"));
   }
}
```

结果：
```java
testData
testData
TestData
```

## Primitive

原始类型，例如：ints

```java
import java.util.List;

import com.google.common.primitives.Ints;

public class GuavaTester {

   public static void main(String args[]) {
      GuavaTester tester = new GuavaTester();
      tester.testInts();
   }

   private void testInts() {
      int[] intArray = {1,2,3,4,5,6,7,8,9};

      //convert array of primitives to array of objects
      List<Integer> objectArray = Ints.asList(intArray);
      System.out.println(objectArray.toString());

      //convert array of objects to array of primitives
      intArray = Ints.toArray(objectArray);
      System.out.print("[ ");
      
      for(int i = 0; i< intArray.length ; i++) {
         System.out.print(intArray[i] + " ");
      }
      
      System.out.println("]");
      
      //check if element is present in the list of primitives or not
      System.out.println("5 is in list? " + Ints.contains(intArray, 5));

      //Returns the minimum		
      System.out.println("Min: " + Ints.min(intArray));

      //Returns the maximum		
      System.out.println("Max: " + Ints.max(intArray));

      //get the byte array from an integer
      byte[] byteArray = Ints.toByteArray(20000);
      
      for(int i = 0; i< byteArray.length ; i++) {
         System.out.print(byteArray[i] + " ");
      }
   }
}
```

结果：

```java
[1, 2, 3, 4, 5, 6, 7, 8, 9]
[ 1 2 3 4 5 6 7 8 9 ]
5 is in list? true
Min: 1
Max: 9
0 0 78 32
```

## Math

数学计算库

## Iterators

使用迭代器简化操作

```java
 List<String> list = Lists.newArrayList("Apple","Pear","Peach","Banana");

Predicate<String> condition = new Predicate<String>() {
    @Override
    public boolean apply(String input) {
        return ((String)input).startsWith("P");
    }
};
boolean allIsStartsWithP = Iterators.all(list.iterator(), condition);
System.out.println("all result == " + allIsStartsWithP);
```

all方法的第一个参数是Iterator，第二个参数是Predicate<String>的实现，这个方法的意义是不需要我们自己去写while循环了，他的内部实现中帮我们做了循环，把循环体中的条件判断抽象出来了。

Iterators类中有partition(Iterator iterator, int size)和 paddedPartition(Iterator iterator, int size)两个函数，
它们都是将iterator中的元素以数量为size分成Iterators.size(iterator) / size + (Iterators.size(iterator) % size == 0 ? 0 : 1)组，
唯一的区别是partition当最后一组数量不是size个时，不会补充；而paddedPartition当最后一组数量不是size个时，会填充null，使得最后一组元素数量也为size个。如下:
```java
Iterable<String> wyp = Splitter.on(",").split("w,y,p,h,a");
Iterator<String> iterator = wyp.iterator();
UnmodifiableIterator<List<String>> listUnmodifiableIterator = Iterators.partition(iterator, 3);
while (listUnmodifiableIterator.hasNext()){
     System.out.println(listUnmodifiableIterator.next());
}
```

结果：
```
[w, y, p]
[h, a]
```

