# 工程感悟

### 思想形态和视角
策略：规划团队架构时，需要清晰化模块和概念，并形成共识。
方法：对技术命名
例子：hclm provisioner

### 技术选型和路线图
系统像搭积木，应建立在坚实清晰的模块（砖头）之上。



Code Review

[RAW MATERIALS]
Bad: 
Example: clouddriver review, introduced specific exception handling at VM deletion op. It indicates that a code was designed for generic purpose error handling, was turned to handle specific error. Such error should be handled at a unified layer.

