
@AutoConfigureOrder
@AutoConfigureBefore
@AutoConfigureAfter


| 注解                            | 本质                                          | 描述 |
| ------------------------------- | --------------------------------------------- | ---- |
| @ConditionalOnBean              | @Conditional(OnBeanCondition.class)           |      |
| @ConditionalOnMissingBean       | @Conditional(OnBeanCondition.class)           |      |
| @ConditionalOnSingleCandidate   | @Conditional(OnBeanCondition.class)           |      |
| @ConditionalOnClass             | @Conditional(OnClassCondition.class)          |      |
| @ConditionalOnMissingClass      | @Conditional(OnClassCondition.class)          |      |
| @ConditionalOnProperty          | @Conditional(OnPropertyCondition.class)       |      |
| @ConditionalOnResource          | @Conditional(OnResourceCondition.class)       |      |
| @ConditionalOnCloudPlatform     | @Conditional(OnCloudPlatformCondition.class)  |      |
| @ConditionalOnExpression        | @Conditional(OnExpressionCondition.class)     |      |
| @ConditionalOnJava              | @Conditional(OnJavaCondition.class)           |      |
| @ConditionalOnJndi              | @Conditional(OnJndiCondition.class)           |      |
| @ConditionalOnWebApplication    | @Conditional(OnWebApplicationCondition.class) |      |
| @ConditionalOnNotWebApplication | @Conditional(OnWebApplicationCondition.class) |      |
| @ConditionalOnWarDeployment     | @Conditional(OnWarDeploymentCondition.class)  |      |
|                                 |                                               |      |



