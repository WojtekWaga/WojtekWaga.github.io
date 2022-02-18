---
layout: blog
title: Transaction is a lie
categories: Databases 
---

If you are working with databases you are familiar with such notions as trasnsaction and ACID. It's fair to say that ACID is all about transactions. But do we need them so desparately that some go ad far as to say that you need ACID guarantees to even being called a database?

If you see a nice acronym like SOLID, ACID, CRUD the chances are some of the letter made it there only to make it read nicely and ACID is no exception. Let's decipher it first:

Atomicity - guarantees that everyone can see all the changes you made or none of them. This is important if you make say a money transfer. A transfer consists of two operations: 1. take money from account A, 2. add money to account B. If you make any of them without the other you will either destroy money or create it out of a thin air.

Consistency - transaction brings database from one consistent state to another. For example your company database includes two tables Departments(dept_id, manager), Employee(empl_id, dept_id) and manager needs to be a valid employee whereas dept_id refers to a department. Let's say you want to add a new department "Ops" with a new manager "Joe". You cannot insert into Department as Joe is not present in employees as well as insert into Employees with dept_id = "Ops". Here come transactions to the rescue! Provided your constraints are 