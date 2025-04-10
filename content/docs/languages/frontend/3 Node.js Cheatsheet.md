---
title: "Node.js Cheatsheet"
weight: 3
bookToc: true
---

## JS基础


### Variables & Data Types

`var`/`let`/`const`的区别？

`var`的特点有如下：
- Hoisting: 变量被提升到他们scope的顶部，可以在未声明变量时访问，此时访问的值为`undefined`
- 在函数内声明的变量为`function-scoped`，只能在函数内部访问；任何函数外部声明为`global-scope`；但在`if (block-scope)`内声明时，可以在外部访问
- 可以在同一个scope重复声明，会被重新赋值

`let`的特点有如下：
- 循环中，`if` 语句中声明时，`block-scope`，只在这个块内有效
- 无Hoisting，提前访问会报错`ReferenceError`
- 不能重复声明，重复声明会报错`SyntaxError`

`const`的特点如下：
- 类似`let`的`block-scope`的特性
- 被赋予`const`类型，则无法重新赋值`reassigned`，但objects的properties，arrays中elements可以被更新

`Primitive Types`：`string`, `number`, `boolean`, `null`, `undefined`

`Reference Types`: `object`, `array`, `function`


{{< hint info >}}
`null`表示值不存在，说明变量存在，但是变量没有一个特定的值，如用于链表的EOF

`undefined`表示未定义，如函数被声明但是没有被初始化，如object的properties不存在，如函数没有参数，函数没有返回值

```javascript
var a;
console.log(a); // Output: undefined

var a = null;
console.log(a); // Output: null
```
{{</hint>}}

### Operators

算数运算符：`+`, `-`, `*`, `/`, `%`  
赋值运算符：`=`, `+=`, `-=`, `*=`, `/=`  
比较运算符：`==`, `===`, `!=`, `!==`, `>`, `<`, `>=`, `<=`  
逻辑运算符：`&&`, `||`, `!`

{{< hint info >}}
`==`比较时会将2个值转化为同一类型，`===`会比较类型
```javascript
0 == '0'; // true because '0' is converted to 0

0 === '0'; // false because the types are different (number vs string)
``` 
{{</hint>}}

### Control Flow

```javascript
// if...else example
var number = 5;
if (number > 0) {
    console.log("Number is positive");
} else {
    console.log("Number is non-positive");
}

// switch example
var day = 3;
switch (day) {
    case 1:
        console.log("Monday");
        break;
    case 2:
        console.log("Tuesday");
        break;
    case 3:
        console.log("Wednesday");
        break;
    default:
        console.log("Another day");
}

// for loop example
for (var i = 0; i < 3; i++) {
    console.log("For loop iteration:", i);
}

// while loop example
var count = 0;
while (count < 3) {
    console.log("While loop count:", count);
    count++;
}

// do...while loop example
var doCount = 0;
do {
    console.log("Do-while loop count:", doCount);
    doCount++;
} while (doCount < 3);
```

### Functions

函数声明：`function functionName(parameters) { ... }`
箭头函数：`(parameters) => { ... }` or `parameter => expression`
函数可以作为其他函数的参数

### Arrays

数组声明：`const arrayName = [element1, element2, ...];`
访问元素：`arrayName[index]`
数据方法：`push()`, `pop()`, `shift()`, `unshift()`, `splice()`, `slice()`, `concat()`, `indexOf()`, `includes()`, `forEach()`, `map()`, `filter()`

```javascript
// Starting with an initial array
let numbers = [1, 2, 3, 4, 5];

// push() - Adds one or more elements to the end of an array
numbers.push(6);
console.log("After push:", numbers);

// pop() - Removes the last element from an array
numbers.pop();
console.log("After pop:", numbers);

// shift() - Removes the first element from an array
numbers.shift();
console.log("After shift:", numbers);

// unshift() - Adds one or more elements to the beginning of an array
numbers.unshift(0);
console.log("After unshift:", numbers);

// splice() - Changes the contents of an array by removing or replacing existing elements and/or adding new elements
numbers.splice(2, 1, 9); // At index 2, remove 1 element and add 9
console.log("After splice:", numbers);

// slice() - Returns a shallow copy of a portion of an array
let sliced = numbers.slice(1, 3);
console.log("Sliced array:", sliced);

// concat() - Merges two or more arrays
let moreNumbers = [7, 8, 9];
let combined = numbers.concat(moreNumbers);
console.log("After concat:", combined);

// indexOf() - Returns the first index at which a given element can be found
console.log("Index of 9:", combined.indexOf(9));

// includes() - Determines whether an array includes a certain value
console.log("Includes 0:", combined.includes(0));

// forEach() - Executes a function for each array element
console.log("forEach:");
combined.forEach(item => console.log(item));

// map() - Creates a new array with the results of calling a function for every array element
let doubled = combined.map(x => x * 2);
console.log("After map (doubled):", doubled);

// filter() - Creates a new array with all elements that pass the test implemented by the provided function
let filtered = doubled.filter(x => x % 2 === 0);
console.log("Filtered even numbers:", filtered);
```

### Objects

声明：`const objName = { key1: value1, key2: value2, ... };`
访问属性Properties：`objName.key` / `objName['key']`
增加/修改属性Properties：`objName.newKey = value` / `objName['newKey'] = value`

Methods: Functions defined as object properties.

### Classes

```JS
// Define a class named 'Person'
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }

    // Method to introduce the person
    introduce() {
        console.log(`Hello, my name is ${this.name} and I am ${this.age} years old.`);
    }
}

// Create an instance of the class
const person1 = new Person('Alice', 30);
person1.introduce();  // Output: Hello, my name is Alice and I am 30 years old.
```

继承：
```JS
// Extend the Person class with a new class called 'Student'
class Student extends Person {
    constructor(name, age, course) {
        super(name, age);  // Call the parent class constructor
        this.course = course;
    }

    // Extend the introduction method to include the student's course
    introduce() {
        super.introduce(); // Call the introduce method from the parent class
        console.log(`I am studying ${this.course}.`);
    }
}

// Create an instance of the Student class
const student1 = new Student('Bob', 20, 'Computer Science');
student1.introduce();  // Output: Hello, my name is Bob and I am 20 years old.
                        // I am studying Computer Science.
```

Getters/Setters 实现对属性访问的控制
```
class Employee extends Person {
    constructor(name, age, position) {
        super(name, age);
        this.position = position;
    }

    get job() {
        return this.position;
    }

    set job(newPosition) {
        this.position = newPosition;
    }
}

// Create an instance of the Employee class
const employee1 = new Employee('Carol', 40, 'Manager');
console.log(employee1.job);  // Output: Manager
employee1.job = 'Director';
console.log(employee1.job);  // Output: Director
```

### Error Handling
```JS
function processFile() {
    try {
        // Simulate file reading error
        throw new Error("Failed to read file");
    } catch (error) {
        console.log("Error:", error.message);
        return false;  // Indicate that the processing failed
    } finally {
        console.log("Cleaning up resources...");  // Always runs
    }
}

processFile();
```

### DOM
```JS
document.getElementById('addButton').addEventListener('click', function() {
    const list = document.getElementById('list');
    const newItem = document.createElement('li');
    newItem.textContent = 'New Item';
    list.appendChild(newItem);
});
```

### Asynchronous Programming

**Callback**
```JS
function fetchDataWithCallback(callback) {
    setTimeout(() => { // Simulate a network request
        callback(null, "Data received (Callback)");
    }, 1000);
}

fetchDataWithCallback((error, data) => {
    if (error) {
        console.log("Error:", error);
    } else {
        console.log(data);
    }
});
```

**Promise**
```JS
function fetchData() {
    return new Promise((resolve, reject) => {
        resolve("Data fetched successfully");  // resolve() function in a JavaScript Promise determines the value that is passed to the .then() handler
        console.log("This will still run after resolve");
    });
}

fetchData().then(data => console.log(data));  // Output: Data fetched successfully
```

{{< hint info >}}
Callback和Promise典型区别

Callback Hell
```JS
getData(function(a){
    parseData(a, function(b){
        filterData(b, function(c){
            transformData(c, function(d){
                displayData(d, function(e){
                    // Finally, handle the result 'e'
                    console.log(e);
                }, errorHandler);
            }, errorHandler);
        }, errorHandler);
    }, errorHandler);
}, errorHandler);

function errorHandler(err) {
    console.log('Error:', err);
}
```

只需要最后一个catch，能捕获到之前任意阶段的异常
```
getData()
    .then(parseData)
    .then(filterData)
    .then(transformData)
    .then(displayData)
    .catch(errorHandler);
```
{{</hint>}}

## Setup

包管理器：`npm --version`

创建项目：`npm init`

安装包：`npm install <package-name>`，依赖会被自动添加到 `package.json` 文件的 `dependencies` 列表中

运行：`node  xxx.js`

自定义脚本运行：`npm start`，需要在 `package.json` 文件中定义
```js
"scripts": {
  "start": "node app.js",
  "test": "echo \"Error: no test specified\" && exit 1"
}
```

## Module

可以从模块中导出函数、变量、类等。有两种主要的导出方式：命名导出和默认导出

命名导出：
```js
// file: math.js
export const pi = 3.14159;
export function add(x, y) {
  return x + y;
}
```
默认导出：
```js
// file: User.js
export default class User {
  constructor(name) {
    this.name = name;
  }
}
```
导入命名导出：
```js
// 导入单个命名导出
import { add } from './math.js';

// 导入多个命名导出
import { add, pi } from './math.js';
```
导入默认导出：在导入一个默认导出的模块时，你可以给它指定任何名字，而不需要知道原始模块中定义的名称。这是因为每个模块只能有一个默认导出，所以当你导入时，默认导出的引用是明确的
```js
import Person from './User.js';
```
导入全部：
```js
import * as Math from './math.js';
console.log(Math.add(1, 2));
```