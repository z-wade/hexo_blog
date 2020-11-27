---
title: IGList Diff算法
date: 2020-11-27 11:45:41
tags: iglist、iOS
categories: 算法
---

## 名词作用解释

* `IGListEntry`： 用于记录数据在新数组和老数组出现的情况
* `table` ：存放所有的 Entry，key-value
* `newResultsArray` ：存到new数据的元素 Entry
* `oldResultsArray` ：存到old数据的元素 Entry
* `newArry` 存放着原始新数据的array
* `oldArry` 存放着原始老数据的array

## Entry类

```
struct IGListEntry {
    /// 在OldArray出现的次数
    NSInteger oldCounter = 0;
    /// 在NewArray出现的次数
    NSInteger newCounter = 0;
    /// 在OldIndex的位置，如果是遍新newArray的话，会往这个栈添加NSNotFound
    stack<NSInteger> oldIndexes;
    /// 标志是否要更新
    BOOL updated = NO;
};
```

## 算法流程

### 第一步

遍历 `newArry`，创建 newResultsArray 里面存放着 IGListRecord，每个元素的newCounter大于0，并向oldIndexs栈添加`NSNotFound`

```
    vector<IGListRecord> newResultsArray(newCount);
    for (NSInteger i = 0; i < newCount; i++) {
        id<NSObject> key = IGListTableKey(newArray[i]);
        IGListEntry &entry = table[key];
        entry.newCounter++;

        // add NSNotFound for each occurence of the item in the new array
        entry.oldIndexes.push(NSNotFound);

        // note: the entry is just a pointer to the entry which is stack-allocated in the table
        newResultsArray[i].entry = &entry;
    }
```

### 第二步：

遍历 oldArray，创建 oldResultsArray 里面存放着 IGListRecord，每个元素的 oldCounter大于0，并且oldIndexs的top为
当前元素在oldArray的index
```
 vector<IGListRecord> oldResultsArray(oldCount);
    for (NSInteger i = oldCount - 1; i >= 0; i--) {
        id<NSObject> key = IGListTableKey(oldArray[i]);
        IGListEntry &entry = table[key];
        entry.oldCounter++;

        // push the original indices where the item occurred onto the index stack
        entry.oldIndexes.push(i);

        // note: the entry is just a pointer to the entry which is stack-allocated in the table
        oldResultsArray[i].entry = &entry;
    }
```

### 第三步

1. 遍历 newResultArray
2. 如果当前 oldIndexes pop出来的位置是小于oldCount的话，即表示该元素在oldArray也存在。
3. 分别从 newArray[i] (注：这是原始数据，并不是上面的newResultArray) 和oldArray[originalIndex] (同样) 取出元素。进行 diff 比较，如果两个不等，设置 updated = YES
4. 如果 newCount 和 oldCount 且 originalIndex != NSNotFound (为什么第三步不用这个来判断？？)。设置 newResultArray和 oldResultArray 里面的 Record 的 index 互为对方的位置;

```
for (NSInteger i = 0; i < newCount; i++) {
        IGListEntry *entry = newResultsArray[i].entry;

        // grab and pop the top original index. if the item was inserted this will be NSNotFound
        NSCAssert(!entry->oldIndexes.empty(), @"Old indexes is empty while iterating new item %li. Should have NSNotFound", (long)i);
        const NSInteger originalIndex = entry->oldIndexes.top();
        entry->oldIndexes.pop();

        if (originalIndex < oldCount) {
            const id<IGListDiffable> n = newArray[i];
            const id<IGListDiffable> o = oldArray[originalIndex];
            switch (option) {
                case IGListDiffPointerPersonality:
                    // flag the entry as updated if the pointers are not the same
                    if (n != o) {
                        entry->updated = YES;
                    }
                    break;
                case IGListDiffEquality:
                    // use -[IGListDiffable isEqualToDiffableObject:] between both version of data to see if anything has changed
                    // skip the equality check if both indexes point to the same object
                    if (n != o && ![n isEqualToDiffableObject:o]) {
                        entry->updated = YES;
                    }
                    break;
            }
        }
        if (originalIndex != NSNotFound
            && entry->newCounter > 0
            && entry->oldCounter > 0) {
            // if an item occurs in the new and old array, it is unique
            // assign the index of new and old records to the opposite index (reverse lookup)
            newResultsArray[i].index = originalIndex;
            oldResultsArray[originalIndex].index = i;
        }
    }
```

### 处理Delete的数据

遍历 `oldResultArray` ，index 为`NSNotFound`设置为删除

```
    for (NSInteger i = 0; i < oldCount; i++) {
        deleteOffsets[i] = runningOffset;
        const IGListRecord record = oldResultsArray[i];
        // if the record index in the new array doesn't exist, its a delete
        if (record.index == NSNotFound) {
            addIndexToCollection(returnIndexPaths, mDeletes, fromSection, i);
            runningOffset++;
        }

        addIndexToMap(returnIndexPaths, fromSection, i, oldArray[i], oldMap);
    }
```

offset的示意图(橙色表示会被删除）

![image](http://note.youdao.com/yws/res/12408/931E6B2D7CCF4DC286CBCEC7B697BACF)

### Insert、Update、Move处理

#### 添加判断逻辑
在newResultArray 中，代码的判断 record.index == NSNotFound ，那就属于要被添加进去的

#### 更新判断逻辑
同样看下面代码，之前设置了update直接添加到update结果里面就行了。


#### 移动的判断逻辑：

在上面会记录了删除后和插入后的偏移位置，如果  oldIndex - deleteOffset + insertOffset 计算后和当前的位置是一样的，就表示不用移动。就好比：["1", "2", "3"] => [""2, "3"]，2的这个位置在新数组里面是 0 ，他的oldIndex = 1, deleteOffset也是1，经过删除后，就是0，即不用移动。

```
for (NSInteger i = 0; i < newCount; i++) {
        insertOffsets[i] = runningOffset;
        const IGListRecord record = newResultsArray[i];
        const NSInteger oldIndex = record.index;
        // add to inserts if the opposing index is NSNotFound
        if (record.index == NSNotFound) {
            addIndexToCollection(returnIndexPaths, mInserts, toSection, i);
            runningOffset++;
        } else {
            // note that an entry can be updated /and/ moved
            if (record.entry->updated) {
                addIndexToCollection(returnIndexPaths, mUpdates, fromSection, oldIndex);
            }

            // calculate the offset and determine if there was a move
            // if the indexes match, ignore the index
            const NSInteger insertOffset = insertOffsets[i];
            const NSInteger deleteOffset = deleteOffsets[oldIndex];
            if ((oldIndex - deleteOffset + insertOffset) != i) {
                id move;
                if (returnIndexPaths) {
                    NSIndexPath *from = [NSIndexPath indexPathForItem:oldIndex inSection:fromSection];
                    NSIndexPath *to = [NSIndexPath indexPathForItem:i inSection:toSection];
                    move = [[IGListMoveIndexPath alloc] initWithFrom:from to:to];
                } else {
                    move = [[IGListMoveIndex alloc] initWithFrom:oldIndex to:i];
                }
                [mMoves addObject:move];
            }
        }

        addIndexToMap(returnIndexPaths, toSection, i, newArray[i], newMap);
    }

```

### 最后生成结果

主要把update的变成 insert和delete
```
    // convert move+update to delete+insert, respecting the from/to of the move
    // 遍历 又要moves和update的值，（前面设置update的时候是还没delete和insert），所以后面会有可能产生 update + move
    const NSInteger moveCount = moves.count;
    for (NSInteger i = moveCount - 1; i >= 0; i--) {
        IGListMoveIndexPath *move = moves[i];
        if ([filteredUpdates containsObject:move.from]) {
            [filteredMoves removeObjectAtIndex:i];
            [filteredUpdates removeObject:move.from];
            [deletes addObject:move.from];
            [inserts addObject:move.to];
        }
    }
    
    // iterate all new identifiers. if its index is updated, delete from the old index and insert the new index
    // 同理，把从老数数的Map里面，取出要更新的，变成insert和delete
    for (id<NSObject> key in [_oldIndexPathMap keyEnumerator]) {
        NSIndexPath *indexPath = [_oldIndexPathMap objectForKey:key];
        if ([filteredUpdates containsObject:indexPath]) {
            [deletes addObject:indexPath];
            [inserts addObject:(id)[_newIndexPathMap objectForKey:key]];
        }
    }
```

##### 扩展阅读

* DeepDiff库（diff算法的另一种实现）
* https://medium.com/flawless-app-stories/a-better-way-to-update-uicollectionview-data-in-swift-with-diff-framework-924db158db86
* https://xiaozhuanlan.com/topic/6921308745  Wagner–Fischer算法
