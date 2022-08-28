## client端使用pinctrl过程的情景分析_基于IMX6ULL

参考资料：

* Linux 5.x内核
  * Documentation\devicetree\bindings\pinctrl\pinctrl-bindings.txt
  * arch/arm/boot/dts/stm32mp151.dtsi
  * arch/arm/boot/dts/stm32mp157-100ask-pinctrl.dtsi  
  * arch/arm/boot/dts/stm32mp15xx-100ask.dtsi
  * drivers\pinctrl\stm32\pinctrl-stm32mp157.c
  * drivers\pinctrl\stm32\pinctrl-stm32.c

* Linux 4.x内核
  * Documentation\pinctrl.txt
  * Documentation\devicetree\bindings\pinctrl\pinctrl-bindings.txt
  * arch/arm/boot/dts/imx6ull-14x14-evk.dts
  * arch/arm/boot/dts/100ask_imx6ull-14x14.dts
  * drivers\pinctrl\freescale\pinctrl-imx6ul.c
  * drivers\pinctrl\freescale\pinctrl-imx.c

#### 1.2 pinctrl

假设芯片上有多个pin controller，那么这个设备使用哪个pin controller？

这需要通过设备树来确定：

* 分析设备树，找到pin controller
* 对于每个状态，比如default、init，去分析pin controller中的设备树节点
  * 使用pin controller的pinctrl_ops.dt_node_to_map来处理设备树的pinctrl节点信息，得到一系列的pinctrl_map
  * 这些pinctrl_map放在pinctrl.dt_maps链表中
  * 每个pinctrl_map都被转换为pinctrl_setting，放在对应的pinctrl_state.settings链表中

![image-20210505182828324](E:/QRS/doc_and_source_for_drivers/IMX6ULL/doc_pic/06_Pinctrl/pic/06_Pinctrl/19_pinctrl_maps.png)

#### 1.3 pinctrl_map和pinctrl_setting

设备引用pin controller中的某个节点时，这个节点会被转换为一些列的pinctrl_map：

* 转换为多少个pinctrl_map，完全由具体的驱动决定
* 每个pinctrl_map，又被转换为一个pinctrl_setting
* 举例，设备节点里有：`pinctrl-0 = <&state_0_node_a>`
  * pinctrl-0对应一个状态，会得到一个pinctrl_state
  * state_0_node_a节点被解析为一系列的pinctrl_map
  * 这一系列的pinctrl_map被转换为一系列的pinctrl_setting
  * 这些pinctrl_setting被放入pinctrl_state的settings链表

![image-20210505182324076](E:/QRS/doc_and_source_for_drivers/IMX6ULL/doc_pic/06_Pinctrl/pic/06_Pinctrl/20_dt_to_map.png)

### 2. client节点的pinctrl构造过程

#### 2.1 函数调用

```
really_probe
	pinctrl_bind_pins
		dev->pins = devm_kzalloc(dev, sizeof(*(dev->pins)), GFP_KERNEL);
		
		dev->pins->p = devm_pinctrl_get(dev);
							pinctrl_get
								create_pinctrl(dev);
									ret = pinctrl_dt_to_map(p);
									
                                    for_each_maps(maps_node, i, map) {
	                                    ret = add_setting(p, map);
                                    }
		
		dev->pins->default_state = pinctrl_lookup_state(dev->pins->p,
					PINCTRL_STATE_DEFAULT);			
```



##### really_probe 函数分析

```c
	/* If using pinctrl, bind pins now before probing */
	ret = pinctrl_bind_pins(dev);	/* 绑定引脚 */
```

##### pinctrl_bind_pins 函数分析

```c
	dev->pins->p = devm_pinctrl_get(dev);	/* 获得pinctrl结构体 */

	/* 查找有无default_state */
	dev->pins->default_state = pinctrl_lookup_state(dev->pins->p,PINCTRL_STATE_DEFAULT); 

    /* 查找有无init_state */   
	dev->pins->init_state = pinctrl_lookup_state(dev->pins->p,PINCTRL_STATE_INIT);

	ret = pinctrl_select_state(dev->pins->p,dev->pins->default_state);
	} else {
		ret = pinctrl_select_state(dev->pins->p, dev->pins->init_state);
	}

	/* 查找有无sleep_state */ 
	dev->pins->sleep_state = pinctrl_lookup_state(dev->pins->p,PINCTRL_STATE_SLEEP);

	/* 查找有无idle_state */ 
	dev->pins->idle_state = pinctrl_lookup_state(dev->pins->p,PINCTRL_STATE_IDLE);


	pinctrl_select_state
    /* 选择相应的状态 */
    switch (setting->type) {
    case PIN_MAP_TYPE_MUX_GROUP:
         ret = pinmux_enable_setting(setting);
         break;
    case PIN_MAP_TYPE_CONFIGS_PIN:
    case PIN_MAP_TYPE_CONFIGS_GROUP:
         ret = pinconf_apply_setting(setting);
         break;

```

##### devm_pinctrl_get 函数分析

```c
	struct pinctrl *p;
	
p = devm_pinctrl_get(dev);   /* 获得pinctrl结构体 */
	p = pinctrl_get(dev);
		p = find_pinctrl(dev);
				list_for_each_entry(p, &pinctrl_list, node)
				if (p->dev == dev)
					return p;

		return create_pinctrl(dev);		/* 第一次find_pinctrl返回为空，调用此函数 */
```

##### create_pinctrl 涉及结构体分析

```c
	struct pinctrl *p;
/* 涉及到的结构体 */
struct pinctrl {
	struct list_head node;
	struct device *dev;
	struct list_head states;
	struct pinctrl_state *state;
	struct list_head dt_maps;
	struct kref users;
};
	struct pinctrl_maps *maps_node;
/* 涉及到的结构体 */
struct pinctrl_maps {
	struct list_head node;
	struct pinctrl_map const *maps;
	unsigned num_maps;
};
	struct pinctrl_map const *map;
/* 涉及到的结构体 */
struct pinctrl_map {
	const char *dev_name;
	const char *name;
	enum pinctrl_map_type type;
	const char *ctrl_dev_name;
	union {
		struct pinctrl_map_mux mux;
		struct pinctrl_map_configs configs;
	} data;
};
```

##### create_pinctrl 函数分析

```c
	ret = pinctrl_dt_to_map(p);	

	/* pinctrl_dt_to_map分析 */
	/* For each defined state ID */
	for (state = 0; ; state++) {
		/* Retrieve the pinctrl-* property */
		propname = kasprintf(GFP_KERNEL, "pinctrl-%d", state);
		prop = of_find_property(np, propname, &size);
    }
	/* Determine whether pinctrl-names property names the state */
	/* 获得pinctrl-names属性名字*/
	ret = of_property_read_string_index(np, "pinctrl-names",state, &statename);
	/* 查找引脚配置节点 */
	for (config = 0; config < size; config++)
		np_config = of_find_node_by_phandle(phandle);
		/* 解析节点 */
		ret = dt_to_map_one_config(p, statename, np_config);


	ret = add_setting(p, map);
	/* add_setting分析 */
	/* 根据情况创建setting */
	switch (map->type) {
	case PIN_MAP_TYPE_MUX_GROUP:
		ret = pinmux_map_to_setting(map, setting);
		break;
	case PIN_MAP_TYPE_CONFIGS_PIN:
	case PIN_MAP_TYPE_CONFIGS_GROUP:
		ret = pinconf_map_to_setting(map, setting);
		break;


```

##### dt_to_map_one_config 函数分析

```c
	const struct pinctrl_ops *ops;
	struct pinctrl_dev *pctldev;
	struct pinctrl_map *map;

	pctldev = get_pinctrl_dev_from_of_node(np_pctldev);	/* 从节点中获取pinctrl_dev */

	ops = pctldev->desc->pctlops;
	/* 调用pinctrl_ops中的dt_node_to_map函数 */
	ret = ops->dt_node_to_map(pctldev, np_config, &map, &num_maps);		

```

##### ops->dt_node_to_map （imx_dt_node_to_map）函数分析

```c
	struct pinctrl_map *new_map;
	
struct pinctrl_map {
	const char *dev_name;
	const char *name;
	enum pinctrl_map_type type;
	const char *ctrl_dev_name;
	union {
		struct pinctrl_map_mux mux;
		struct pinctrl_map_configs configs;
	} data;
};
	/* 根据驱动设备树中的pinctrl名字在左边找到相应的group */
	grp = imx_pinctrl_find_group_by_name(info, np->name);
	/* 创建n+1个new_map结构体，保存数据 （n等于每个组引脚的个数）*/
	/* 第一个结构体保存group的信息，其余的保存引脚信息 */
	new_map[0].type = PIN_MAP_TYPE_MUX_GROUP;	/* 表示保存的组信息 */
	new_map[0].data.mux.function = parent->name;	/* 这组引脚要作用的功能 */
	new_map[0].data.mux.group = np->name;	/* 用到的组的名字 */
	
	new_map++;
	new_map[j].type = PIN_MAP_TYPE_CONFIGS_PIN;
	new_map[j].data.configs.group_or_pin = pin_get_name(pctldev, grp->pins[i].pin);/* 每个引脚的名字 */
	new_map[j].data.configs.configs = &grp->pins[i].pin_conf.pin_memmap.config;	  /* 保存配置信息 */				new_map[j].data.configs.num_configs = 1;
	j++;

```

