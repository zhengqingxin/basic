# async / await

## 划重点：
* async 函数返回的是一个 Promise 对象。



## 错误写法
* 当几个方法没有依赖关系，下面写法是错的，会造成阻塞：
```js
async function mount() {
  const result1 = await fetch('a.json');
  const result2 = await fetch('b.json');
  const result3 = await fetch('c.json');

  render(result1, result2, result3);
}
```
正确：
```js
async function mount() {
  const result = await Promise.all(
    fetch('a.json'),
    fetch('b.json'),
    fetch('c.json')
  );

  render(...result);
}
```