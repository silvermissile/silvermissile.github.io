``` mermaid
graph LR
  idea(Intelej)
  open(打开)
  idea --> search(查找)
  idea --> open(打开)
  idea --> edit(编辑)
  idea --> run(运行)






%% Set edges to be curved (try monotoneX for a nice alternative)
%%linkStyle default interpolate basis
linkStyle default interpolate monotoneX


  subgraph
    open(打开)
    open --> cmd+e(cmd+e:打开最近文件)
    open --> some_user(cmd+e:打开最近文件)
    open --> some_user(cmd+e:打开最近文件)
    open --> some_user(cmd+e:打开最近文件)
    open --> some_user(cmd+e:打开最近文件)
  end
```

- 当我想从两个方向上延伸的时候，子图


``` mermaid
graph TB
    c1-->a2
    subgraph one
    a1-->a2
    end
    subgraph two
    b1-->b2
    end
    subgraph three
    c1-->c2
    end
```

1. jupyterhub 能起来，配上公网IP能访问，新启动一个都没关系。
2. 网络还是用host模式
