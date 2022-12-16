# pnpm menorepo demo project

https://blog.csdn.net/astonishqft/article/details/124823381

## Get Start
```
pnpm init
```

### 全局的公共依赖包
全局安装`father`

```
pnpm i -Dw father
```

pnpm 提供了 -w, --workspace-root 参数，可以将依赖包安装到工程的根目录下，作为所有 package 的公共依赖
```
pnpm install react -w
```

开发依赖，可以加上 -D，表示这是开发依赖，会装到 pacakage.json 中的 devDependencies 中，比如：
```
pnpm install rollup -wD
```

### 给某个package单独安装指定依赖
给 pkg1 安装一个依赖包，比如 axios
```
pnpm add axios --filter @farando/monorepo1
```

执行 pkg1 下的 scripts 脚本
```
pnpm build --filter @farando/monorepo1
pnpm build --filter "./packages/**"
```

> 注意的是，--filter 参数跟着的是package下的 package.json 的 name 字段，并不是目录名。

### 模块之间的相互依赖
在 pkg1 中引用 pkg2
```
pnpm install @farando/monorepo2 -r --filter @farando/monorepo1
```

> 设置依赖版本的推荐用 `workspace:*`，这样可以保持依赖的版本是工作空间里最新版本，不需要每次手动更新

## 只允许pnpm
根`package.json`中添加：
```
{
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

## Release工作流
### 配置changesets
安装
```
pnpm add -Dw @changesets/cli
```

初始化
```
pnpm changeset init
```

修改配置文件`.changeset\config.json`如下：
```
{
  "$schema": "https://unpkg.com/@changesets/config@2.2.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [["@farando/*"]],
  "access": "public",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": [],
  "___experimentalUnsafeOptions_WILL_CHANGE_IN_PATCH": {
    "onlyUpdatePeerDependentsWhenOutOfRange": true
  }
}
```

- changelog: changelog 生成方式
- commit: 不要让 changeset 在 publish 的时候帮我们做 git add
- linked: 配置哪些包要共享版本
- access: 公私有安全设定，内网建议 restricted ，开源使用 public
- baseBranch: 项目主分支
- updateInternalDependencies: 确保某包依赖的包发生 upgrade，该包也要发生 version upgrade 的衡量单位（量级）
- ignore: 不需要变动 version 的包
- ___experimentalUnsafeOptions_WILL_CHANGE_IN_PATCH: 在每次 version 变动时一定无理由 patch 抬升依赖他的那些包的版本，防止陷入 major 优先的未更新问题

### Config
根package.json中加入
```
"scripts": {
  "changeset": "changeset",
  "version-packages": "changeset version",
  "release": "pnpm build && pnpm release:only",
  "release:only": "changeset publish --registry=https://registry.npmjs.com/"
},
```

### 如何使用changesets
```
# 1-1 进行了一些开发...
# 1-2 提交变更集
pnpm changeset

# 1-3 提升版本
# changeset version
pnpm version-packages 

# 1-4 发包
# pnpm build && pnpm changeset publish --registry=...
pnpm release 
```

## 规范代码提交

### 安装
```
pnpm install -wD commitizen cz-conventional-changelog
```

### 配置
根package.json增加：
```
"scripts": {
  "commit": "cz"
}
```

### pnpm commit
> 使用 `pnpm commit` 来代替 `git commit` 进行代码提交


## commitlint && husky

### 安装
```
pnpm install -wD @commitlint/cli @commitlint/config-conventional husky
```

### Config
根目录下增加 `commitlint.config.js`:
```
module.exports = { extends: ['@commitlint/config-conventional'] };
```

根 package.json 中增加一条 script:
```
"scripts": {
  "postinstall": "husky install"
}
```

执行`pnpm install`，会在根目录下创建一个 .husky 目录。

执行如下命令新增一个husky的hook
```
npx husky add .husky/commit-msg 'npx --no-install commitlint --edit "$1"'
```

## 代码规范检查
### 安装
```
pnpm install -wD eslint lint-staged @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

### Config
在根成根目录下添加 .eslintrc.js
```
module.exports = {
  'parser': '@typescript-eslint/parser',
  'plugins': ['@typescript-eslint'],
  'rules': {
    'no-var': 'error',// 不能使用var声明变量
    'no-extra-semi': 'error',
    '@typescript-eslint/indent': ['error', 2],
    'import/extensions': 'off',
    'linebreak-style': [0, 'error', 'windows'],
    'indent': ['error', 2, { SwitchCase: 1 }], // error类型，缩进2个空格
    'space-before-function-paren': 0, // 在函数左括号的前面是否有空格
    'eol-last': 0, // 不检测新文件末尾是否有空行
    'semi': ['error', 'always'], // 在语句后面加分号
    'quotes': ['error', 'single'],// 字符串使用单双引号,double,single
    'no-console': ['error', { allow: ['log', 'warn'] }],// 允许使用console.log()
    'arrow-parens': 0,
    'no-new': 0,//允许使用 new 关键字
    'comma-dangle': [2, 'never'], // 数组和对象键值对最后一个逗号， never参数：不能带末尾的逗号, always参数：必须带末尾的逗号，always-multiline多行模式必须带逗号，单行模式不能带逗号
    'no-undef': 0
  },
  'parserOptions': {
    'ecmaVersion': 6,
    'sourceType': 'module',
    'ecmaFeatures': {
      'modules': true
    }
  }
};
```

package.json 中增加
```
"lint-staged": {
    "*.ts": [
      "eslint --fix",
      "git add"
    ]
}
```

husky 中增加 pre-commit 校验
```
npx husky add .husky/pre-commit "npx --no-install lint-staged"
```


