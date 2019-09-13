
EncodingIdMapper 
put 方法：

```java

long eId = encode( inputId );

dataCache.set( nodeId, eId );

groupCache.set( nodeId, group.id() );

candidateHighestSetIndex.offer( nodeId );
```

dataCache.set( nodeId, eId );
```java
DynamicLongArray.set(long index, long value )

at( index ).set( index, value );

1. at
{DynamicNumberArray<N extends NumberArray<N>>

    @Override
    public N at( long index )
    {
        if ( index >= length() )
        {
            synchronizedAddChunk( index );
            // 扩容 new OffHeapLongArray( length, defaultValue, base );
        }

        int chunkIndex = chunkIndex( index );
        return chunks[chunkIndex];
    }
}

at 方法先看情况给 chunks 扩容，实际上就是新建一个 OffHeapLongArray ，放到 chunks 数租，然后返回一个。实际上就是返回当前的顶点应该所在的 LongArray。

2. set
{OffHeapLongArray
    
    @Override
    public void set( long index, long value )
    {
        UnsafeUtil.putLong( addressOf( index ), value );
    }
    
    addressOf(index)
    {
        index = rebase( index ); // index - base;
        if ( index < 0 || index >= length )
        {
            throw new ArrayIndexOutOfBoundsException( "Requested index " + index + ", but length is " + length );
        }
        // 
        return address + (index << shift); // 在当前位置左移三位，也就是乘以8，因为保存一个 long 要 8位？
    }
    putLong:
    unsafe.putLong( address, value );
    
}


```

dataCache 的set逻辑比较清楚了，dataCache 是一个 DynamicLongArray ，里面有一个 chunks = OffHeapLongArray[] ,
每个 OffHeapLongArray 长度是一百万，数据增多了就新建 OffHeapLongArray。
然后通过 Unsafe 的方法赋值，因为 ne4j 的 id 是连续的，所以直接移动八位就是下一个数据的存储地址。然后 putLong 放进去。
这里我们其实有一些疑问，现在其实是将顶点的值 value 放到了它的 id 对应的位置上，我们要查一个 id 对应的值只需要找到对应地址即可，
我们需要查询一个 value 对应的 id 还是没法查，所以我们接下来要研究一下怎么查。

看 get 方法，看名字就知道这是一个二分查找，二分查找有个前提是数据是排好序的，然后不断得到中间的值和现有的值对比。
```java
private long binarySearch( Object inputId, int groupId )
{
    long low = 0;
    long high = highestSetIndex;
    // 得到加密后的 value 
    long x = encode( inputId );
    // 这个方法得到 x 的基数
    int rIndex = radixOf( x );
    
    // for 循环中，rIndex 和 sortBuckets 二维数组的 第一行的值作比较，如果满足了条件，就取对应的第二行和下一个第二行的值作为 low 和 high。
    // 根据这个逻辑我们大概能判断出来，sortBuckets 中放的是分位点，第一行放的是 rIndex 分位点，第二行放的是 index 的分位点，而且 index 对应的值一个个是排好序的，不然没法二分查找。
    // 
    for ( int k = 0; k < sortBuckets.length; k++ )
    {
        if ( rIndex <= sortBuckets[k][0] )//bucketRange[k] > rIndex )
        {
            low = sortBuckets[k][1];
            high = (k == sortBuckets.length - 1) ? highestSetIndex : sortBuckets[k + 1][1];
            break;
        }
    }

    long returnVal = binarySearch( x, inputId, low, high, groupId );
    if ( returnVal == ID_NOT_FOUND )
    {
        low = 0;
        high = highestSetIndex;
        returnVal = binarySearch( x, inputId, low, high, groupId );
    }
    return returnVal;
}


private long binarySearch( long x, Object inputId, long low, long high, int groupId )
{
    while ( low <= high )
    {
        // 中值点 low 和 high 都是 index
        long mid = low + (high - low) / 2;//(low + high) / 2;
        
        // trackerCache 能根据值 index 得到 真正的 index ？
        
        long dataIndex = trackerCache.get( mid );
        if ( dataIndex == ID_NOT_FOUND )
        {
            return ID_NOT_FOUND;
        }
        // 查找 value，dataCache 是我们上面放数据的 DynamicLongArray ，根据 index 能查到 值。
        long midValue = dataCache.get( dataIndex );
        switch ( unsignedDifference( clearCollision( midValue ), x ) )
        {
        case EQ:
            // We found the value we were looking for. Question now is whether or not it's the only
            // of its kind. Not all values that there are duplicates of are considered collisions,
            // read more in detectAndMarkCollisions(). So regardless we need to check previous/next
            // if they are the same value.
            boolean leftEq = mid > 0 && unsignedCompare( x, dataValue( mid - 1 ), CompareType.EQ );
            boolean rightEq = mid < highestSetIndex && unsignedCompare( x, dataValue( mid + 1 ), CompareType.EQ );
            if ( leftEq || rightEq )
            {   // OK so there are actually multiple equal data values here, we need to go through them all
                // to be sure we find the correct one.
                return findFromEIdRange( leftEq ? mid - 1 : mid, rightEq ? mid + 1 : mid, midValue, inputId, x, groupId );
            }
            // This is the only value here, let's do a simple comparison with correct group id and return
            return groupOf( dataIndex ) == groupId ? dataIndex : ID_NOT_FOUND;
        case LT:
            low = mid + 1;
            break;
        default:
            high = mid - 1;
            break;
        }
    }
    return ID_NOT_FOUND;
}
```

整个查询到这里感觉很乱，我们先理一下，我们要在 dataCache 中查找某个 x，首先通过 radixOf 方法得到一个 rIndex，再通过 rIndex 在 sortBuckets 数组中找到 high 和 low，
在第二个方法中，high 和 low，可以通过 trackerCache 得到 dataIndex，然后 dataCache.get( dataIndex ) 得到了 value。

其实熟悉排序算法的话，第二个方法并不难明白。在有的排序算法中，排序算法涉及到交换，而我们保存在内存的数据进行交换是很麻烦的，所以我们再加一个数组记录数据下标，每次不交换数据只交换下标。
这种方式本身不复杂，只是多了一层，可能比较绕，我们用一个例子梳理一下。 
假如：要排序的三个数 data[2,4,1], 我们先记下 index[0,1,2]，然后进行排序，以冒泡排序为例：
    1. 比较 data[index[0]], data[index[1]],无需交换，然后比较 data[index[1]], data[index[2]],发现 4比1大，然后交换index为[0,2,1],
    2. 然后继续比较 data[index[0]], data[index[1]],发现2大于1，交换 index 为 [2,0,1] ，
    3. 得到的排序结果是 data[2,3,1], index[2,0,1] 。
如果是别的排序方法类似，例如快速排序，我们先记下 data[2,5,1,3,0],index[0,1,2,3,4]，首先所有的数据和 data[index[0]] 比较，得到index[2,4,0,1,3]，然后继续得到 index[4,2,0,3,1].

排序好了以后，我们需要进行查找操作，而二分查找也需要先根据index找到对应位置的数据，
例如上面5个数据我要查找 3 ,首先 left=0,right=4, mid =2，而 data[index[2]]=2,小于3，所以 left=2+1,mid=3 ，data[index[3]]=3,查找到了对应的索引是 3,

所以我们可以猜想这里也是这个道理 dataCahce 排序的时候并没有交换数据二是通过 trackerCache 交换了对应的索引。
而要查找 dataCahce 的数据，首先在得到 mid ，这个 mid 实际上是数据按照大小的排名，然后从 trackerCache 得到排名为 mid 的索引，
然后调用 dataCache 得到排名为 mid 的数据，和已有的数据进行比较。

我们看看 trackerCache 注释：
```java
// Ordering information about values in dataCache; the ordering of values in dataCache remains unchanged.
// in prepare() this array is populated and changed along with how dataCache items "move around" so that
// they end up sorted. Again, dataCache remains unchanged, only the ordering information is kept here.
// Each index in trackerCache points to a dataCache index, where the value in dataCache contains the
// encoded input id, used to match against the input id that is looked up during binary search.
```

注释说的很清楚，trackerCache 是 dataCache 的排序信息，dataCache 的数据是不变的， prepare() 执行的时候 trackerCache 生成并且随着 dataCache 排序而改变，记录 dataCache 的顺序，
trackerCache 的每个索引指向 dataCache 的一个位置，where 放置了 加密的 inputId ，在 binarysearch 用来匹配输入的 inputId。那重点就是 prepare() 方法。

trackerCache 的问题解决了，sortBuckets 大概是什么意思？dataCache 的长度可能有几十亿，即便是二分查找，也要很久，所以我们是否可以大概确定一个上下限呢？其实就是桶排序，或者基数排序。
例如：data[1,2,3,4,5,6,7,8,9,10] 我们要查找 9的位置，是否可以先把它分成 [1,2,3,4,5],[6,7,8,9,10] 两个，记下他们的上下限 [0，5],[1,10]。来一个数据先缩小一下它的范围，再进行二分查找？

而对于存在内存中的数据，需要进行基数排序，然后进行快速排序是有一定要求的。

我们看一下需要看一下 encode 和 radixOf 的代码，

encode:Encodes String into a long with very small chance of collision

```java
@Override
public long encode( Object s )
{
    int[] val = encodeInt( (String) s ); 
    return (long) val[0] << 32 | val[1] & UPPER_INT_MASK; // 连接 val[0] 和 val[1]
}

private int[] encodeInt( String s )
{
    // 将 String 变成bytes
    int inputLength = s.length();
    byte[] bytes = new byte[inputLength];
    for ( int i = 0; i < inputLength; i++ )
    {
        bytes[i] = (byte) ((s.charAt( i )) % 127);
    }
    
    // 调用 remap, 将 bytes[i] 转化为 reMap[bytes[i]]
    reMap( bytes, inputLength );
    
    // encode 长度小于 7 的 简单编码
    if ( inputLength <= encodingThreshold )
    {
        
        return simplestCode( bytes, inputLength );
    }
    int[] codes = new int[numCodes];
    for ( int i = 0; i < numCodes; )
    {
        codes[i] = getCode( bytes, inputLength, 1 );
        codes[i + 1] = getCode( bytes, inputLength, inputLength - 1 );
        i += 2;
    }
    int carryOver = lengthEncoder( inputLength ) << 1;
    int temp = 0;
    for ( int i = 0; i < numCodes; i++ )
    {
        temp = codes[i] & FOURTH_BYTE;
        codes[i] = codes[i] >>> 8 | carryOver << 24;
        carryOver = temp;
    }
    return codes;
}
private int lengthEncoder( int length )
{
    if ( length < 32 )
    {
        return length;
    }
    else if ( length <= 96 )
    {
        return length >> 1;
    }
    else if ( length <= 324 )
    {
        return length >> 2;
    }
    else if ( length <= 580 )
    {
        return length >> 3;
    }
    else if ( length <= 836 )
    {
        return length >> 4;
    }
    else
    {
        return 127;
    }
}

// reMap 方法就是给 bytes 做一个一一隐射，将 bytes[i] 转化为 reMap[bytes[i]]，0<=bytes[i]<=255
private void reMap( byte[] bytes, int inputLength )
{
    for ( int i = 0; i < inputLength; i++ )
    {
        if ( reMap[bytes[i]] == -1 )
        {
            synchronized ( this )
            {
                if ( reMap[bytes[i]] == -1 )
                {
                    reMap[bytes[i]] = (byte) (numChars++ % 256);
                }
            }
        }
        bytes[i] = reMap[bytes[i]];
    }
}

// 
private int[] simplestCode( byte[] bytes, int inputLength )
{
    int[] codes = new int[]{0, 0};
    codes[0] = max( inputLength, 1 ) << 25; 长度左移 25位，inputLength_0000000000
    codes[1] = 0;
    for ( int i = 0; i < 3 && i < inputLength; i++ )
    {
        codes[0] = codes[0] | bytes[i] << ((2 - i) * 8);  bytes[i] 左移 16,8,0 位后和已有的 codes[0] 取或，
    }
    for ( int i = 3; i < 7 && i < inputLength; i++ )
    {
        codes[1] = codes[1] | (bytes[i]) << ((6 - i) * 8);bytes[i] 左移 16,8,0 位后和已有的 codes[1]取或，
    }
    return codes;
}

private int getCode( byte[] bytes, int inputLength, int order )
{
    long code = 0;
    int size = inputLength;
    for ( int i = 0; i < size; i++ )
    {
        //code += (((long)bytes[(i*order) % size]) << (i % 7)*8);
        long val = bytes[(i * order) % size];
        for ( int k = 1; k <= i; k++ )
        {
            long prev = val;
            val = (val << 4) + prev;//% Integer.MAX_VALUE;
        }
        code += val;
    }
    return (int) code;
}
```

encode 太复杂了，我们知道是尽量保证不重复的long即可，编码的前 8位 是长度编码，后面的是数据编码，中间留了一个0.

radixOf : Calculates and keeps radix counts. Uses a {@link RadixCalculator} to calculate an integer radix value from a long value.

```java
protected static final int RADIX_BITS = 24; // 24 位
protected static final long LENGTH_BITS = 0xFE000000_00000000L; // 1111_1110_00000000000000000 (56个0)
protected static final int LENGTH_MASK = (int) (LENGTH_BITS >>> (64 - RADIX_BITS)); // 右移40位 还剩24位  1111_111_00000000000000 (17 个零)
protected static final int HASHCODE_MASK = (int) (0x00FFFF00_00000000L >>> (64 - RADIX_BITS)); 右移40位 还剩24位  0000_0000_1111_1111_1111_1111
/**
  * Radix optimized for strings encoded into long by {@link StringEncoder}.
  */
 public static class String extends RadixCalculator
 {
     @Override
     public int radixOf( long value )
     {
         int index = (int) (value >>> (64 - RADIX_BITS));  RADIX_BITS = 24 右移40位 还剩高位24个位，
         index = ((index & LENGTH_MASK) >>> 1) | (index & HASHCODE_MASK);  //  (FE00 & index) | (00FFFF & index) 其实就是取 index 的前7位和后16位。也就是去掉第17位
         return index; // 整个逻辑就是先去掉低40位变成24位 index，然后去掉第17位。
     }
 }
```
编码最后去掉第17位是因为第17位是 0 没有意义。

这里大概知道用了什么原理，接下来我们 prepare 方法，看看 sort 的过程。我们大概已经知道使用了 基数排序和快速排序。

ParallelSort 的 run 方法，首先是 sortRadix

```java
// 实际上就是将数据按照 thread 数量分为几个桶，
private long[][] sortRadix() throws InterruptedException
{
    long[][] rangeParams = new long[threads][2];
    int[] bucketRange = new int[threads];
    // 初始化的工作，初始化应该主要就是得到几个分桶。
    Workers<TrackerInitializer> initializers = new Workers<>( "TrackerInitializer" );
    // sortBuckets 保存分桶信息，一共两行，第一行是捅的 index ，第二行是数据的 index
    sortBuckets = new long[threads][2];
    
    long dataSize = highestSetIndex + 1;
    long bucketSize = dataSize / threads;
    long count = 0;
    long fullCount = 0;
    progress.started( "SPLIT" );
    //遍历所有数据 radixIndex， 每个基数对应的数据的个数，基数 = 先去掉低40位变成24位 index，然后第17位变为0。
    for ( int i = 0, threadIndex = 0; i < radixIndexCount.length && threadIndex < threads; i++ )
    {
        // 已找到的的数据 ，也就是基数 > bucketSize
        if ( (count + radixIndexCount[i]) > bucketSize )
        {
            // 现在要根据线程数将整个基数分成几个桶， bucketRange 保存分位点位置，
            bucketRange[threadIndex] = count == 0 ? i : i - 1; 
            // rangeParams 的第一行是截至目前数据总量
            rangeParams[threadIndex][0] = fullCount;
            if ( count != 0 )
            {
                // rangeParams 的第二行是当前数据量
                rangeParams[threadIndex][1] = count;
                fullCount += count;
                progress.add( count );
                count = radixIndexCount[i];
            }
            else
            {
                rangeParams[threadIndex][1] = radixIndexCount[i];
                fullCount += radixIndexCount[i];
                progress.add( radixIndexCount[i] );
            }
            initializers.start( new TrackerInitializer( threadIndex, rangeParams[threadIndex],
                    threadIndex > 0 ? bucketRange[threadIndex - 1] : -1, bucketRange[threadIndex],
                    sortBuckets[threadIndex] ) );
            threadIndex++;
        }
        else
        {
            count += radixIndexCount[i];
        }
        if ( threadIndex == threads - 1 || i == radixIndexCount.length - 1 )
        {
            bucketRange[threadIndex] = radixIndexCount.length;
            rangeParams[threadIndex][0] = fullCount;
            rangeParams[threadIndex][1] = dataSize - fullCount;
            initializers.start( new TrackerInitializer( threadIndex, rangeParams[threadIndex],
                    threadIndex > 0 ? bucketRange[threadIndex - 1] : -1, bucketRange[threadIndex],
                    sortBuckets[threadIndex] ) );
            break;
        }
    }
    progress.done();

    // In the loop above where we split up radixes into buckets, we start one thread per bucket whose
    // job is to populate trackerCache and sortBuckets where each thread will not touch the same
    // data indexes as any other thread. Here we wait for them all to finish.
    Throwable error = initializers.await();
    long[] bucketIndex = new long[threads];
    int i = 0;
    for ( TrackerInitializer initializer : initializers )
    {
        bucketIndex[i++] = initializer.bucketIndex;
    }
    if ( error != null )
    {
        throw new AssertionError( error.getMessage() + "\n" + dumpBuckets( rangeParams, bucketRange, bucketIndex ),
                error );
    }
    return rangeParams;
}

```

TrackerInitializer 的 start 启动一个线程，我们看看 run 方法。

```java
public void run()
{
    for ( long i = 0; i <= highestSetIndex; i++ )
    {
        // 计算基数，也就是在 整个基数桶 中的位置
        int rIndex = radixCalculator.radixOf( comparator.dataValue( dataCache.get( i ) ) );
        
        // 看看这个数是否属于这个桶。
        if ( rIndex > lowRadixRange && rIndex <= highRadixRange )
        {
            // rangeParams 保存了当前线程在桶中的上下限。
            long trackerIndex = rangeParams[0] + bucketIndex++;
            assert tracker.get( trackerIndex ) == -1 :
                    "Overlapping buckets i:" + i + ", k:" + threadIndex + ", index:" + trackerIndex;
            
            // 这个设置到 tracker 中。每个 trackerIndex 对应一个 i，每个 i 也对应一个 trackerIndex。
            tracker.set( trackerIndex, i );
            
            // 如果到达了上界。
            if ( bucketIndex == rangeParams[1] )
            {
                result[0] = highRadixRange;
                result[1] = rangeParams[0];
            }
        }
    }
}
```

TrackerInitializer 的 run 方法作用就是在 tracker 设置 trackerIndex 和 i 的对应关系， trackerIndex 是 i 的基数在桶中的位置 + 当前 bucket 的 Index bucketIndex 是递增的。

然后我们看 SortWorker 排序。关键逻辑在 partition 中。

```java
private long partition( long leftIndex, long rightIndex, long pivotIndex )
{
    // left 和 right ，right 为倒数第二个
    long li = leftIndex;
    long ri = rightIndex - 2;
    long pi = pivotIndex;
    long pivot = clearCollision( dataCache.get( tracker.get( pi ) ) );
    
    // save pivot in last index，先把 pivot 放到最后
    tracker.swap( pi, rightIndex - 1 );
    long left = clearCollision( dataCache.get( tracker.get( li ) ) );
    long right = clearCollision( dataCache.get( tracker.get( ri ) ) );
    while ( li < ri )
    {
        // 左边和 pivot 比较
        if ( comparator.lt( left, pivot ) )
        {   
            // this value is on the correct side of the pivot, moving on。 左边的比 pivot 小，不用管，直接看下一个。
            left = clearCollision( dataCache.get( tracker.get( ++li ) ) );
        }
        else if ( comparator.ge( right, pivot ) )
        {   // this value is on the correct side of the pivot, moving on 。右边的比 pivot 大，不用管，下一个
            right = clearCollision( dataCache.get( tracker.get( --ri ) ) );
        }
        else
        {   // this value is on the wrong side of the pivot, swapping， 右边的比 pivot 小，左边的比 pivot 大，交换一下。
            tracker.swap( li, ri );
            long temp = left;
            left = right;
            right = temp;
        }
    }
    long partingIndex = ri;
    if ( comparator.lt( right, pivot ) )
    {
        partingIndex++;
    }
    // restore pivot
    tracker.swap( rightIndex - 1, partingIndex );
    return partingIndex;
}
```

这是快速排序的实现方法。
截至目前，其实还有一点没解决，就是 tracker ，tracker 中存放了 trackerIndex 和 index 的键值对，里面是怎么存储的其实值得我们研究。
