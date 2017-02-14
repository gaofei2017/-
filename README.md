# 封装轻量级 jQuery

## 描述 jq 对象的本质与选择模块的封装
### 1.循环的封装
        function func ( arr, method ) {
            ///
        }

        func( arr, 'max' );

        我们现在要封装的循环应该是一个通用的循环. 是描述各种类型的循环代码.

        封装的循环就是将通用的内容抽取出来, 然后将共同的东西封装.
        for ( var i = 0; i < arr.length; i++ ) {
            ... 不同点 ...
        }
        如果要封装就将中间的抽取出来
        function each( arr, statement ) {
            for ( var i = 0; i < arr.length; i++ ) {
                eval( statement );  // 一般是在这里遍历 第 i 项, 的元素 arr[ i ]
            }
        }
        在可操作上会很麻烦:
        1> 程序员必须知道这个函数体中使用遍历时的各种变量.
        2> 如何循环体的代码非常的复杂, 那么书写变得会很麻烦.

        利用 new Function 来实现数组的排序.
        var arr = [ ... ]
        for ( var i = 0; i < arr.length - 1; i++ ) {
            for ( var j = 0; j < arr.length - 1 - i; j++ ) {
                if ( arr[ j ] > arr[ j + 1] ) {
                    ...
                }
            }
        }

        var sort = new Function ( 'arr', 'for ( var i = 0; i < arr.length - 1; i++ ) {' +
                                         'for ( var j = 0; j < arr.length - 1 - i; j++ ) {' + 
                                         ... );
                                        

        要解决, 
        1> 保证不用知道内部变量的名字就可以使用 遍历的数据, 知道遍历的第 几 项, 以及该项的数据是什么即可
        2> 需要提供一段可以在内部执行的代码. 而且要求应该非常容易编写. 

        function each( arr, func ) {
            for ( var i = 0; i < arr.length; i++ ) {
                func( i, arr[ i ] ); // 调用
            }
        }
        那么在编写函数的时候就可以约定, 参数表示的含义.
        在使用的时候, 就可以利用
        each(arr, function ( i, v ) {
            ...
        })

        回调函数有专有的名词 callback, 在有了 对 jq 中, 和 ES5 中数组方法的分析后, 我们应该知道参数的顺序 v, i
        
        function each( arr, callback ) {
            for ( var i = 0; i < arr.length; i++ ) {
                callback( arr[ i ], i ); // 调用
            }
        }

###2. 引入链式编程后的问题
    1> 不能继续链式
    2> each 方法有点尴尬,只能遍历数组

###3.解决尴尬
#### 1)each 需要可以遍历数组的同时还可以遍历对象
    -> jq 中 each, map 与 ES5 中扩展的 forEach 和 map 的异同
        each: 
        	数组, 对象,回调函数的参数是i,v return false跳出循环, 返回值是遍历的对象本身, this 是当前元素
        map: 
        	数组, 对象, 确定是否要返回数据, 返回新数组.没有return 返回空数组[]
        ES5:
        	只能操作数组，回调函数的参数都是v i
            forEach: 不能跳出, 不能用 this, 不能返回数据等
            map:返回一个新数组，没有return则返回一个undefined
    -> each
        
#### 2)如何判断对象是一个数组或伪数组
    // 只需要判断 length 属性即可
    // arr
    1> 是不是一个数组
        toString.call() -> '[object Object]' 形式给出
        instanceof 方法也可以实现, 但是在 html 嵌套的页面中会有问题

        Object.prototype.toString.call( arr ) == '[object Array]'
    
    2> 是不是伪数组?
        1) 必须含有 length 属性
            'length' in arr
        2) 判断他是不是数字
            typeof length == 'number'
        3) 因此长度需要 非负
            length >= 0
        
        var length = 'length' in arr && arr.length;
        return typeof length === 'number' && length >= 0;

    function isArrayLike( obj ) {
        if ( Object.prototype.toString.call( obj ) == '[object Array]' ) {
            return true;
        }
        var length = 'length' in arr && arr.length;
        return typeof length === 'number' && length >= 0;
    }

###4. map 方法
    功能与 each 几乎一样, 但是没有考虑跳出循环, 没有考虑 this, 
    考虑 回调函数的返回值作为 返回新数组的元素.
    在 jq 中 map 方法中回调函数的参数是 v, i;
    而 在 each 方法中回调函数的参数是 i, v

###5. 链式编程的特点
    一般语法结构:
        func1( ... )
            .func2( ... )
            .func3( ... )
            ...
    链式编程有一个非常重要的特征, 就是前面一个函数执行完, 则将其结果交给下一个函数继续执行
    因此需要注意的是, 我们返回的对象应该具有下一个方法, 才可以继续调用.

    在 jq 中知道所有的方法都是 jq 对象的或 $ 函数的

    策略: 将所有的方法都应该挂载到 $ 函数 或 jq 对象中

    jq 的本质就是 一个带有很多方法的伪数组, 数组元素就是 各种 DOM 对象.

    思考:
        1> 如何将 jq 对象转换成 DOM 对象 
            $( dom )[ 0 ]
        2> 如何判断页面中是否有某元素
            if ( $( ... ).length > 0 ) {}
        3> 猜测: jq 就是一个伪数组.

    链式编程
        构造一个存储dom元素的 对象, 同时带有很多的方法, 让每一个方法都是在处理我们
        每一个 DOM 元素, 然后 返回 this 即可实现无限的链式编程了.
