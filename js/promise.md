# Promise

* Promise 新建后就会立即执行。
* 调用resolve或reject并不会终结 Promise 的参数函数的执行。
* resolve(anotherPromise),会等 anotherPromise 有结果后，外面的 promise 才会执行then/catch。
* Promise 会吃掉错误。
* 用catch, ps：catch后会返回一个 resolve 的promise
* all/race 中 promise 数组中如果自己处理了异常，就不会调用外层的catch

## 实例

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
### 请求设置缓存
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

## 资料：
[http://es6.ruanyifeng.com/#docs/promise](http://es6.ruanyifeng.com/#docs/promise)

[https://davidwalsh.name/promises](https://davidwalsh.name/promises)