# 开发时

/Plugins
	/xxx/TypeScript
/TypeScript
	/Config
		/TsConfig
		/EslintConfig
		/...一些公共配置项
	/Packages
		/ReactUMG
		/PuertsCli
		/PuertsEditor
	/Project
		/LyraEditor
		/LyraGame
	puerts.config.json
	pnpm-workspace.yaml

Packages主要放开发/运行时的依赖包
Project为运行时Gameplay脚本和Editor
# 脚本打包/开发打包
/Plugins
	/GameFeatures
		/ShooterGame
			/Content
				/JavaScript
					....js
/Content
	/JavaScript
		/node_modules
		/react-umg
		/Project
			/LyraUE
			/LyraGame
			/LyraEditor
		/Packages
			...
		/...


### puerts-pnp.config.json
标识包名，源代码位置，js的输出位置，以及tags
```json
{
    "packages": [
        {
            "name": "@lyra-ts/lyra-ue",
            "folder": "TypeScript/Project/LyraUE",
            "target": "Content/JavaScript/Project/LyraUE",
            "tags": ["uclass_extends"]
        },
        {
            "name": "@lyra-ts/lyra-game",
            "folder": "TypeScript/Project/LyraGame",
            "target": "Content/JavaScript/Project/LyraGame",
            "tags": []
        },
        {
            "name": "@lyra-ts/lyra-editor",
            "folder": "TypeScript/Project/LyraEditor",
            "target": "Content/JavaScript/Project/LyraEditor",
            "tags": ["uclass_extends"]
        },
        {
            "name": "@lyra-ts/puerts-editor",
            "folder": "TypeScript/Packages/PuertsEditor",
            "target": "Content/JavaScript/PuertsEditor"
        },
        {
            "name": "react-umg",
            "folder": "TypeScript/Packages/ReactUMG",
            "target": "Content/JavaScript/react-umg",
        }
    ]
}
```
tags是用来标识这个包的构建标识/编辑器标识，比如uclass_extends就是指Puerts的继承引擎类是否可以扫描这个包内的ts来生成蓝图

# PuertsPnp
参考 https://yarnpkg.com/features/pnp

