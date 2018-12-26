## VDOM patch
> VDOM patch 在目前主流的前端框架广泛使用。顾名思义就是最小化地操作DOM，给DOM打补丁。Vue和react等框架里面使用的diff算法可能不一致，但是最后都能达到Patch的效果。网上对React的diff算法讲得比较多，所以这里我们以Vue为例，看一下Vue的Patch过程。
## 源码解读
```javascript
// patch.js => updateChildren

  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
  }


```
> 大体上分为两个阶段：1、while循环，主要进行move操作和create操作。2、循环之后的操作，批量删除和批量插入。
> while循环阶段用于判断oldVnode是否为undefined（如果在newCh里面不满足后面的条件但是在oldCh找到），头头相等、尾尾相等、头尾相等、尾头相等、找到对应key移动（上面判断undefined操作来源于此）、未找到对应key创建这几种情况。
## 测试demo
> 这里选取Vue 2.5.17版本。测试代码如下
```javascript

const list1 = [{
  name: '111',
  key: '111',
  show: false
}, {
  name: '222',
  key: '222',
  show: false
}, {
  name: '333',
  key: '333',
  show: true
}, {
  name: '444',
  key: '444',
  show: true
}, {
  name: '555',
  key: '555',
  show: true
}, {
  name: '666',
  key: '666',
  show: false
}]

const list2 = [{
  name: '222',
  key: '222',
  show: true
}, {
  name: '444',
  key: '444',
  show: false
}, {
  name: '333',
  key: '333',
  show: true
}, {
  name: '111',
  key: '111',
  show: true
}, {
  name: '555',
  key: '555',
  show: false
}, {
  name: '777',
  key: '777',
  show: true
}]

/* start */

const v3 = new Vue({
  template: `
    <div>
      <div v-for="(item, index) in list" :key="item.key" v-if="item.show">
        {{item.name}}
      </div>
    </div>
  `,
  data () {
    return {
      list: list1
    }
  },
  mounted () {
    setTimeout(() => {
      /* 这里可以打断点，源码里patch.js patchVnode和 updateChildren 方法内打上断点 */
      this.list = list2
    }, 1000)
  }
})

v3.$mount(`#app3`)

```
## 详解过程
### 初始Vnodes
![setTimeout之后的Vnodes图示](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/1.png)
*N节点代表comment节点*
*注意：操作开始，Vue会将未渲染的节点标记为comment节点。所以如果你第三方框架想要支持类似Vue的patch操作，需要在DOM层实现comment节点。*
> 这里我们要关注的地方在于：1、实现DOM规范node节点的childNodes属性和children属性，childNodes这个类数组包含着最后渲染到视图上的子节点（包含comment节点），children包含所有的element元素。2、两次Vnode数组newCh和OldCh。
![调试器中新旧两次Vnodes](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/1-1.png)
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/1-2.png)
### 第一次循环操作
![第一次循环操作](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/2.png)
> 由于第一次循环操作不符合头头、尾尾、头尾、尾头相同的情况，创建一个oldKeyToIdx存储key的对象，供所有的循环操作使用。newStartVnode在oldKeyToIdx找不到对应的Vnode，所以创建一个新的element并插入到oldStartVNode的ele之前，newStartIndex后移一位，进入下一次循环。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/2-1.png)
### 第二次循环操作
![第二次循环操作](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/3.png)
> 由于第二次循环操作符合头头相等的情况，所以oldStartIndex和newStartIndex都后移一位，进入下一次循环。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/3-1.png)
### 第三次循环操作
![第三次循环操作](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/4.png)
> 由于第三次操作不符合头头、尾尾、头尾、尾头相同的情况。但是newStartVnode在oldKeyToIdx找到对应的key，并且是相同的element，复用oldVnode的ele，将ele插入到oldStartVnode的ele之前。同时将在oldCh中找到的相同key的Vnode设为undefined，newStartIndex后移一位，进入下次循环。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/4-1.png)
### 第四次循环操作
![第四次循环操作](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/5.png)
> 由于第四次循环操作不符合头头、尾尾、头尾、尾头相同的情况。newStartVnode在oldKeyToIdx找不到对应的key，所以新建一个新的element，插入到oldStartVnode的ele之前（insertedVnodeQueue用于递归操作，保存内部的elment）。newStartIndex后移一位，进入下次循环。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/5-1.png)
### 第五次循环操作
![第五次循环操作](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/6.png)
> 由于第五次循环操作符合头头相等的情况，所以oldStartIndex和newStartIndex都后移一位，进入下一次循环。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/6-1.png)
### 第六次循环操作
![第六次循环操作](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/7.png)
> 由于第六次循环操作oldStartVnode是undefined，所以直接跳过，oldStartIndex后移一位。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/7-1.png)
### 第七次循环操作
![第七次循环操作](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/8.png)
> 由于第七次循环操作不符合头头、尾尾、头尾、尾头相同的情况。newStartVnode在oldKeyToIdx找不到对应的key，所以新建一个新的element，插入到oldStartVnode的ele之前（insertedVnodeQueue用于递归操作，保存内部的elment）。newStartIndex后移一位，进入下次循环。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/8-1.png)
### 跳出循环
![跳出循环](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/9.png)
> 由于NewStartIndex > NewEndIndex，所以跳出循环。开始批量删除或者添加操作，这里批量删除oldCh中oldStartIndex到oldEndIndex之间vnode对应的ele。这里删除 444、555、N这三个节点。
![调试器中parentElm](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/9-1.png)
### 最终结果
![最终结果](https://raw.githubusercontent.com/LeeeeeeM/blog/master/2018-12-26/image/10.png)
## 后记
> 在我们快应用中引入了Vue框架，针对diff算法改进了我们的DOM实现。使得Vue能够在快应用平台完美使用。