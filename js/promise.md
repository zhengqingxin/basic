# Promise

## 划重点
* Promise 新建后就会立即执行。
* 调用resolve或reject并不会终结 Promise 的参数函数的执行。
* resolve(anotherPromise),会等 anotherPromise 有结果后，外面的 promise 才会执行then/catch。
* Promise 会吃掉错误（不会影响外部代码执行，node中有 `unhandledRejection` 事件可以捕捉到）
* 用catch, ps：catch后会返回一个 resolve 的promise
* all/race 中 promise 数组中如果自己处理了异常，就不会调用外层的catch

## 实例

### 等待 x 秒
```js
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

### 设置请求超时时间
```js
const p = Promise.race([
  fetch('/resource-that-may-take-a-while'),
  new Promise(function (resolve, reject) {
    setTimeout(() => reject(new Error('request timeout')), 5000)
  })
]);
p.then(response => console.log(response));
p.catch(error => console.log(error));
```
### 请求设置缓存
```js
var userCache = {};

function getUserDetail(username) {
  // In both cases, cached or not, a promise will be returned

  if (userCache[username]) {
  	// Return a promise without the "new" keyword
    return Promise.resolve(userCache[username]);
  }

  // Use the fetch API to get the information
  // fetch returns a promise
  return fetch('users/' + username + '.json')
    .then(function(result) {
      userCache[username] = result;
      return result;
    })
    .catch(function() {
      throw new Error('Could not find user: ' + username);
    });
}
```

### 同步转异步
```js
function makeSyncAsync() {  
  return Promise.resolve().then(function(){
    // execute synchronous code that might throw
    return value;
  });
}
```

### 不要这么写
错误：
```js
return askValue().then(function (value) {
    return a(value);
}).then(function (value) {
    return b.then(function (value) {
        return c(value);
    });
}).then(function (value) {
    return d(value);
});
```
正确：
```js
return askValue().then(function (value) {
    return a(value);
}).then(function (value) {
    return b(value);
}).then(function (value) {
    return c(value);
}).then(function (value) {
    return d(value);
});
```

## 测一测
* 给定两个函数，说出4种写法的输出什么
```js
function foo() {
  console.log('foo');
  return new Promise(resolve => {
    console.log('foo timeout before');
    setTimeout(() => {
      console.log('foo timeout');
      resolve('foo resolved');
    }, 1000);
    console.log('foo timeout after');
  });
}

function bar() {
  console.log('bar');
  return new Promise(resolve => {
    console.log('bar timeout before');
    setTimeout(() => {
      console.log('bar timeout');
      resolve('bar resolved');
    }, 3000);
    console.log('bar timeout after');
  });
}
```

```js
//1
foo().then(function(res) {
  return bar();
});

//2
foo().then(bar);

//3
foo().then(bar()).then(res=>console.log(res));

//4
foo().then(function(res) {
  bar();
});
```
结果：
```js
/*
1 & 2: 没区别，2是1的简写
"foo"
"foo timeout before"
"foo timeout after"
"foo timeout"
"bar"
"bar timeout before"
"bar timeout after"
"bar timeout"
3:如果bar返回一个函数，则会等它，否则直接跳过(这里bar返回了一个Promise，直接跳过)，将 foo 的结果传递给下一个then。
"foo"
"foo timeout before"
"foo timeout after"
"bar"
"bar timeout before"
"bar timeout after"
"foo timeout"
"foo resolved"
"bar timeout"
4:结合3和1
"foo"
"foo timeout before"
"foo timeout after"
"foo timeout"
"bar"
"bar timeout before"
"bar timeout after"
"bar timeout"
*/
```