---
layout: post
title: 'Java의 매개변수 전달방식'
subtitle: 'call-by-value'
author: 'sunyoung.dev'
date: 2019-04-27
header-img: img/in-post/call.jpg
lang: ko
catalog: true
tags:
  - 자바
---

Java는 매개 변수를 전달할 때 언제나 call-by-value 방식입니다. 변수를 넘길 때 변수의 주소값을 넘기는게 아니라 변수에 저장된 값을 복사해 메서드 안에서 사용합니다. 그래서 매개변수의 값은 메서드 안에서만 통용되는 local-variable 입니다.  

간단한 예를 통해 확인해보겠습니다.  

### primitive type의 매개변수

```java
public class BasicTest {
    @Test
    public void test1() {
        int a = 1;

        plusOne(a);

        System.out.println("plusOne 메소드 밖에서 a값 : " + a);
    }

    private void plusOne(int a) {
        a = a+1;

        System.out.println("plusOne 메소드 안에서 a값 : " + a);
    }
}
```

실행 결과  

> plusOne 메소드 안에서 a값 : 2  
> plusOne 메소드 밖에서 a값 : 1   

plusOne 메소드의 매개변수 a에는 caller 메소드(test1)에서 넘긴 a의 값이 복사되어 담겨있을 뿐 입니다. 그래서 메소드 안에서 일어난 변경이 caller에는 전달되지 않습니다. 매개변수로 객체를 전달 받아도 같을까요?  

### Object type의 매개변수

```java
public class BasicTest {
    @Test
    public void test2() {
        Book book = new Book();
        book.setIsbn(123);
        book.setTitle("abc");

        changeBook(book);

        System.out.println("changeBook 밖 : " + book);
    }

    private void changeBook(Book book) {
        Book newBook = new Book();
        newBook.setTitle("abc new");

        book = newBook;

        System.out.println("changeBook 안 : " + book);
    }
}

@Data
class Book {
    int isbn;
    String title;
}
```

실행결과  

> changeBook 안 : Book(isbn=0, title=abc new)  
> changeBook 밖 : Book(isbn=123, title=abc)  

객체를 전달해도 메소드 밖에서 객체가 바뀌지 않는 걸 보니 call-by-value가 맞나봅니다. 그렇다면 메소드 안에서 객체의 상태를 바꿔도 변하지 않을까요?   

### Object type의 매개변수 상태 변경

```java
public class BasicTest {
    @Test
    public void test() {
        Book book = new Book();
        book.setIsbn(123);
        book.setTitle("abc");

        editBook(book);

        System.out.println("editBook 밖 : " + book);
    }

    private void editBook(Book book) {
        book.setTitle("cdf");

        System.out.println("editBook 안 : " + book);
    }
}
```

실행결과  

> editBook 안 : Book(isbn=123, title=cdf)  
> editBook 밖 : Book(isbn=123, title=cdf)

아니 이럴수가 있나요? 분명 메소드의 매개변수 값은 call-by-value로 전달해서 지역적으로 사용될텐데 왜 메소드 안에서 수정한 결과가 메소드 바깥에도 반영되는 걸까요? 이거 정말 call-by-value로 전달되는게 맞습니까?  

결과적으로 말하면 Java에서 매개변수 전달방식은 언제나 call-by-value가 맞습니다. 마지막 예제가 call-by-reference 처럼 보이는 것은 객체 변수가 저장하고 있는 값이 reference 이기 때문입니다.  

### 변수에 저장되는 값
자바에서 primitive type의 변수에 저장되는 값은 value 자체입니다. 반면에 Object type의 변수에 저장되는 값은 그 객체가 저장되어 있는 곳의 주소 즉, reference 입니다.   

그래서 primitive type의 매개변수를 전달할 때는 primitive type 변수가 가지고 있는 값인 value 값 자체가 복사되고 Object type의 매개변수를 전달할 때는 Object type 변수가 가지고 있는 value 값 즉, reference가 복사되어 전달되는 것입니다.  

두 번째 예제의 경우에는 매개변수로 넘겨받은 값(book)에 새로운 객체(newBook)를 할당했으니 매개변수 값(book)에 새로운 객체의 reference 값(newBook의 주소)이 저장됩니다. caller에서 넘겨준 원 객체에 대한 참조는 새로운 객체로 덮어씌워질 뿐 원객체에 대한 변경은 일어나지 않는 것입니다.  

세 번째 예제의 경우에는 매개변수로 넘겨받은 변수 book은 원래 book 객체가 가지고 있는 값인 book값의 주소가 복사되어 담겨있습니다. 그래서 book을 참조로 객체의 상태를 수정하면 원객체의 값이 수정되는 것입니다.  
