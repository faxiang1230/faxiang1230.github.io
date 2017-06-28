链接：https://zhuanlan.zhihu.com/p/19686537

庄表伟曾撰文谈及研发管理的三个提升，由于研发、质量保障、运维三者连接紧密、不分家，所以下面郑昀将其扩展了一下：要『从一个整体来考虑企业的研发管理，应该注重建立一个良性的循环：

    技术能力的提升，主要依靠经验积累，建立企业内部的知识库（如RCA案例库、最佳实践库）与传承体系（促进交流与协作，借助研发活力促进技术能力提升，这个技术能力包括部署、维护、私有云等自动化运维能力）；
    生产效率（而不仅仅是开发效率）的提升，主要依靠科学的数据分析，建立或引进一系列的工具，建立合理的流程与制度（通过提升研发人员、质量保障人员、运维人员能力，激发他们不断改进效率，也很重要）；
    研发活力的提升，促进研发人员积极的交流与分享 （给研发人员松绑，让他们有足够的空余时间，也很重要）；

』

单就研发部门的技术总监（或研发总监，注意不是研发经理或架构师）而言，郑昀定义这个岗位通常要致力于：

    - 横跨各个开发组织的
        - 技术（通用）方案的积极推广
        - 技术（疑难）问题的定位和解决
        - 学习型技术组织的引导和培养
        - 技术工具的（发现或）制造

而研发部门的架构师，则只需要致力于：

    - 横跨各个开发组织的
        - 技术（通用）方案的积极推广
        - 技术工具的（发现或）制造

即可。

具体工作场景举例：

一，技术（通用）方案的积极推广：

    方法：
        找到通用性强的技术问题，抽象业务场景；
        或制定方案，在某开发组织内落地；
        或将A组织的优秀方案复制到B组织；
        或将A公司的优秀方案复制到内部落地；
    例子：
        技术问题举例：
            业务降级
                由于各种线上运维需求，导致部分业务必须停服。几次之后，我们意识到这必须做成功能，随时能通过一个持久化配置中心的控制台界面让某些业务停服而不影响其他业务。
                这就是业务降级解决方案的由来。也因此要求它要扩展为业务降级打包预案，随时可以让一部分业务“批量”降级。

二，技术（疑难）问题的定位和解决

    方法：
        由上级主管发现各个开发小组中的技术问题，尤其是那些线上问题；
        一般来说，开发组组长自己内部解决问题，但上级主管需要判断哪些问题得让研发总监、其他开发组长、架构师等一起商讨解决；
        在这个过程中，形成整个技术团队有事儿一起商量一起解决（而不是各自为战）的氛围。

三，学习型技术组织的引导和培养

    方法：
        部门内有一两个人专门定期组织技术分享讲座
        新人入职后做一次技术分享；
        老人做完一个项目之后做一次技术总结和分享；
        对于部门未来可能遇到的技术难题，提前组织人做课题研究，并做多次技术传道，从浅到深

四，技术工具的（发现或）制造

    方法：
        找到技术团队的痛点；
        找到技术团队的生产效率低的原因；
        抽象业务场景；
        针对性了解其他公司如何解决的，梳理各种方案；
        发现现有开源工具，或组织人员开发工具，制定和验证高可用方案。
    例子：
        自动化测试自动化部署
        持续集成
        定时任务调度和管理
        可靠的异步推送 NotifyServer