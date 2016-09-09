# 安装指引

1. 将源码拉下来

```
git clone git@github.com:ZENGzoe/ZENGzoe.github.io.git zengzoe
```

2. 初始化子模块

```
cd zengzoe
git submodule init
git submodule update
```

3. 安装依赖

```
npm install
```

4. 运行 `hexo s --watch`

访问地址： [http://localhost:4000](http://localhost:4000)

# 创建文章

```
hexo new "my new post"
```

5. 发布文章

```
hexo d
```

6.提交代码

```
git add .
git commit -m "提交"
git push origin hexor
```


