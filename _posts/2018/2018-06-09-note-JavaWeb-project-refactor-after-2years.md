---
title: 记一次时隔两年后的JavaWeb项目重构总结
date: 2018-06-09 11:12:22
tags: ['Java']
categories: ['Java']
---

两年前的2016年，我还没有大学毕业，也才大三下学期，也还有自己的team，一起学习，一起成长，一起技术研究与试炼。不缅怀……当时和自己的team一起开发了[“科技计划项目电子辅助验收及评估平台”](https://gitee.com/Zychaowill/steap)，然而因为team刚成立几个月，其次，项目也比较赶，在时间紧迫的情况下，只能个人保证自己模块不出问题，然后最后再由作为项目负责人的我来审核并集体进行测试。

那么究竟里面的设计有多烂呢？答案就是，有好有坏。在此就不太过多讨论这些了，有兴趣你可以直接去我的[码云Git](https://gitee.com/Zychaowill/steap)看一下这个项目的当时的版本。
<!-- more -->
# **1. 为什么要重构该项目？**

原因有很多，但是最重要的原因应该是：

- 整个系统是围绕着项目状态为发展路线进行展开的，但是项目状态管理上却很混乱，而且扩展起来特别麻烦，几乎不可能。

- 一个通性问题，就是成员技术能力良莠不齐，代码书写以及设计上也会有多多少少的问题。

# **2. 主要针对该项目进行那些重新设计？**

鉴于博主已经工作了，所以大多精力都在工作上，其次也要给自己留一些时间充电，所以时间有限，只针对最重要的一点进行了重新设计，即系统整理的项目状态管理。

- 旧代码中项目状态如何管理的？

旧系统中项目状态是存在系统字典表，项目信息表也有针对项目的状态标记字段。

项目状态大都是直接通过SQL，传入项目编号以及项目的新状态标记，然后进行` UPDATE `操作的。这样造成的项目状态代码不仅分散到各处，而且需要开发人员知道项目所有的状态以及项目当前所处的状态，这样就会出现很多误操作等大量问题。

- 新设计是如何实现项目状态管理的？

new design主要是基于Spring Boot开发（而旧系统开发较早，采用的Struts 2 + Spring + MyBatis，还算兼容，问题不大）。

针对项目状态管理的设计点就是：当项目启动时，自动加载系统字典中的状态类信息并根据`SortNo`进行排序，然后构建不同种类状态树` DictionaryHierarchyTree `，统一放到动态加载的自定义Bean` SystemDataHolder ` 中。而` SystemDataHolder ` 则为系统提供统一的项目状态操作工具，而且我们对项目状态信息一无所知，只需要要简单的调用` SystemDataHoder#nextStatus(Project) ` 即可。

# **3. 新设计核心代码介绍**

针对新设计，核心代码主要分为三个部分：

- a) 加载数据的Service

- b) 动态bean构建

- c) listener的监听

## **3.1 加载数据的Service和动态bean构建**

以下只提供`LoadSystemDataServiceImpl `中的核心代码，其余均进行了省略。而主要核心部分就是 ` LoadSystemDataServiceImpl#initSystemData ` 方法。

该Service主要提供了两个功能，` loadSystemData ` 和 ` reloadSystemData ` ，分别用于项目启动加载系统数据和当修改了系统字典时重新加载系统数据。

```Java
@Slf4j
@Service
public class LoadSystemDataServiceImpl implements LoadSystemDataService, ApplicationContextAware {

	@Autowired
	private SystemDictionaryService systemDictionaryService;

	private ApplicationContext ctx;

	//.......

	@Override
	public void loadSystemData() {
		Map<SystemDictionaryConstants, DictionaryHierarchyTree> systemData = null;
		SystemDictionaryConstants[] dictionaryIndexes = SystemDictionaryConstants.values();
		if (ArrayUtils.isNotEmpty(dictionaryIndexes)) {
			systemData = new HashMap<>(dictionaryIndexes.length);
			for (SystemDictionaryConstants dictionaryIndex : dictionaryIndexes) {
				DictionaryHierarchyTree dictionaryHierarchyTree = systemDictionaryService.getDictionaryHierarchyTree(dictionaryIndex.getCode());
				systemData.put(dictionaryIndex, dictionaryHierarchyTree);
			}
			initSystemData(systemData);
		}
	}

	@Override
	public void reloadSystemData() {
		destorySystemData();
		loadSystemData();
	}

	private void initSystemData(Map<SystemDictionaryConstants, DictionaryHierarchyTree> systemData) {
		log.info("Start loading system data>>>>>>>");

		DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) ctx.getAutowireCapableBeanFactory();
		BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(SystemDataHolder.class);
		beanDefinitionBuilder.addPropertyValue("systemData", systemData);
		beanFactory.registerBeanDefinition("systemDataHolder", beanDefinitionBuilder.getBeanDefinition());

		log.info("Finish loading system data>>>>>>>");
	}

	private void destorySystemData() {
		((DefaultListableBeanFactory) ctx.getAutowireCapableBeanFactory()).removeBeanDefinition("systemDataHolder");

		log.info("Destory system data>>>>>>>");
	}

	//.......
}
```

## **3.2 listener的监听**

主要用于达到项目系统就从DB加载系统字典数据构建字典树。

可以看到，该处调用了上面提供的` LoadSystemDataService#loadSystemData ` 方法。

```Java
@Component
public class LoadSystemDataListener implements CommandLineRunner {

	@Autowired
	private LoadSystemDataService loadSystemDataService;

	@Override
	public void run(String... args) throws Exception {
		loadSystemDataService.loadSystemData();
	}

}
```

## **3.3 项目状态管理应用案例？**

```Java
@RestController
@RequestMapping("/project")
public class ProjectApi {

	@Autowired
	private SystemDataHelper helper;

	@RequestMapping(value = "/action/update")
	public JsonEntity<Project> updateProjectStatus() {
		Project project = new Project();
		project.setStatus(100001);
		project.setProjectName("Machine Learning");
		project.setId("mllib101");
		Project newProject = helper.nextStatus(project);
		return ResponseHelper.of(newProject);
	}
}
```

启动项目，在浏览器中键入` http://localhost:8081/project/action/update ` 进行访问，你会得到如下信息：

![2018-06-09_111011.png](https://upload-images.jianshu.io/upload_images/3631711-551f3d7a85d792bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


因为我们的代码中针对项目设计的项目状态是100001，而项目下一个状态是100002，所以，得到如此结果，就是正确的，也是我们想要的效果，隔离代码与DB的直接访问以及开发人员对过多的信息进行感知。

设计到此结束，其实，解决方案有很多，但是这个只是我采用的，觉得最轻便的一个。有兴趣的话，可以到我得[Github](https://github.com/buildupchao/Steapx)查看重构的完整代码。
