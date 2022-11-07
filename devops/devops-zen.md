## DevOps 道与术


运维活动中的最佳实践形成了平台规范，平台规范必须要通过技术手段落实为工具，仅靠人的自觉是不可靠的。市场上的开源软件，虽然没有形成较完整的解决方案，但是在某项领域内做的不错还是挺多的。e.g. jenkins,gitlab,harbor etc.


DevOps工具链涉及到很多工具，只是单纯的连接到一起往往需要在多个界面间切换。"运维平台"的目标是用科学的工具整合零散的资源管理、规范制度、手工操作，实现“一站式运维”，工程师不需要切换系统，一个平台解决所有事情。

### 构建单元的逻辑划分

Git是现今最流行的版本管理工具，最小的构建单元就是"仓库"(repository). 怎样合理划分构建单元以便管理？

划分方式当然有很多,大多是树形结构，比如:

- 组织->产品线->仓库 (项目管理方式)
- 群组->子群组->仓库（gitlab方式）

## Reference 

1. [如何从零思考设计你的DevOps运维服务体系?](https://www.ytso.com/240624.html)