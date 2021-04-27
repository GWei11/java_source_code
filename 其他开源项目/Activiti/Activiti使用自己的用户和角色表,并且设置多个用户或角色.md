# Activiti 使用自己用户和角色表

在 Activiti6 中还是配套了











可以在 org.activiti.engine.impl.persistence.entity.IdentityLinkEntityManagerImpl#addCandidateGroups(TaskEntity taskEntity, Collection<String> candidateGroups) 这里打上断点，是在这里面执行的添加用户组。