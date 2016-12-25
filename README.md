## Install Hexo

```bash
sudo apt-get install nodejs-legacy npm
sudo npm install -g hexo-cli
```

## Clone website

```bash
git clone https://github.com/jarsonfang/jarsonfang.github.io.git blog
cd blog && git submodule init && git submodule update
```

## Install node_modules

```bash
cd blog && npm install
```

## Preview on localhost

```bash
hexo s
```

## Generate and deploy

```bash
hexo d -g
```
