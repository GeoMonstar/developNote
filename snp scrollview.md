#### 要点：

`UIScrollView是依靠与其子视图（subview）之间的约束来确定ContentSize的大小`，如果不设置好子view的宽高度约束的话，就会造成UISCrollView显示异常。

对于UIScrollView的subview来说，它的leading/trailing/top/bottom的space是相对于UIScrollView的contentSize而不是bounds来确定的，换句话说：UIScrollView与其subview之间相对位置的约束并不会直接用于frame的计算,而是会转化为对ContentSize的计算。(摘自  
:[https://blog.csdn.net/longshihua/java/article/details/78441466](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Flongshihua%2Fjava%2Farticle%2Fdetails%2F78441466))

##### 错误用法

没有指定greenView的 宽高， 无法显示scrollView

```go
    let scrollView = UIScrollView()
    scrollView.delegate = self
    scrollView.backgroundColor = .blue
    view.addSubview(scrollView)
    scrollView.snp.makeConstraints { (make) in
        make.edges.equalToSuperview()
    }

    let greenView = UIView()
    greenView.backgroundColor = .green
    scrollView.addSubview(greenView)
    greenView.snp.makeConstraints { (make) in
        make.leading.trailing.equalToSuperview()
        make.top.equalToSuperview()
        make.bottom.equalToSuperview()
    }
```

2、对subviews指定宽高，让最底部的view设置对scrollView的约束，可显示，但是会有布局异常。

```go
  let scrollView = UIScrollView()
    scrollView.delegate = self
    scrollView.backgroundColor = .blue
    view.addSubview(scrollView)
    scrollView.snp.makeConstraints { (make) in
        make.edges.equalToSuperview()
    }

    let greenView = UIView()
    greenView.backgroundColor = .green
    scrollView.addSubview(greenView)
    greenView.snp.makeConstraints { (make) in
        make.top.leading.equalToSuperview()
        make.width.equalTo(300)
        make.height.equalTo(100)
    }

    let redView = UIView()
    redView.backgroundColor = .red
    scrollView.addSubview(redView)
    redView.snp.makeConstraints { (make) in
        make.leading.equalToSuperview()
        make.top.equalTo(greenView.snp.bottom)
        make.width.equalTo(300)
        make.height.equalTo(1000)
        make.bottom.equalToSuperview() // 不添加底部的约束 则无法滚动
    }
```

##### 正确用法

对scrollView 添加contentView，让subviews添加对contentView的约束，而contentView只需处理contentSize的宽度

```go
    let scrollView = UIScrollView()
    scrollView.delegate = self
    scrollView.backgroundColor = .blue
    view.addSubview(scrollView)
    scrollView.snp.makeConstraints { (make) in
        make.edges.equalToSuperview()
    }
    
    contentView = UIView()
    contentView.backgroundColor = .yellow
    scrollView.addSubview(contentView)
    contentView.snp.makeConstraints { (make) in
        make.edges.equalToSuperview()
        make.width.equalTo(300) 
        // 设置contentSize的 width值
    }
    
    let redView = UIView()
    redView.backgroundColor = .red
    contentView.addSubview(redView)
    redView.snp.makeConstraints { (make) in
        make.leading.top.trailing.equalToSuperview()
        make.height.equalTo(400)
    }
    
    let greenView = UIView()
    greenView.backgroundColor = .green
    contentView.addSubview(greenView)
    greenView.snp.makeConstraints { (make) in
        make.leading.trailing.equalToSuperview()
        make.height.equalTo(600)
        make.top.equalTo(redView.snp.bottom)
        make.bottom.equalToSuperview() 
        // 固定底部约束，设置好contentSize的 height值
    }
```

  
