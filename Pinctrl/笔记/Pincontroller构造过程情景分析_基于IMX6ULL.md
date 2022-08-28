## Pincontroller构造过程情景分析_基于IMX6ULL

imx6ull的构造过程

驱动程序位置：

```
drivers\pinctrl\freescale\pinctrl-imx6ul.c
drivers\pinctrl\freescale\pinctrl-imx.c
```

调用过程：

```c
imx6ul_pinctrl_probe
    imx_pinctrl_probe(pdev, pinctrl_info);
        imx_pinctrl_desc->name = dev_name(&pdev->dev);  
        imx_pinctrl_desc->pins = info->pins;
        imx_pinctrl_desc->npins = info->npins;
        imx_pinctrl_desc->pctlops = &imx_pctrl_ops;   
        imx_pinctrl_desc->pmxops = &imx_pmx_ops;
        imx_pinctrl_desc->confops = &imx_pinconf_ops;
        imx_pinctrl_desc->owner = THIS_MODULE;
		
		ret = imx_pinctrl_probe_dt(pdev, info);    //从设备树保存信息到相应结构体

        ipctl->pctl = devm_pinctrl_register(&pdev->dev,
                            imx_pinctrl_desc, ipctl);
                            
        /* 通过pinctrl_desc（imx_pinctrl_desc）注册生成pinctrl_dev（ipctl->pctl） */
```

### imx_pinctrl_probe_dt 函数分析

```c
imx_pinctrl_dt_is_flat_functions
    /* 此函数可以分辨设备树下有几个功能，在imx6ull中只有一个功能，为imx6ul-evk */
	......
    info->group_index = 0;
	if (flat_funcs) {
		info->ngroups = of_get_child_count(np);
	} else {
		info->ngroups = 0;
		for_each_child_of_node(np, child)	/* 找到最近的一个子节点 */
			info->ngroups += of_get_child_count(child);		/* 统计子节点中的子节点数量。也就是有多少组引脚 */
	}
	......
	if (flat_funcs) {
		imx_pinctrl_parse_functions(np, info, 0);
	} else {
		for_each_child_of_node(np, child)
			imx_pinctrl_parse_functions(child, info, i++);	/* 解析function节点 */
	}

```

### imx_pinctrl_parse_functions 函数分析

```c
	struct imx_pmx_func *func;
	/* 涉及到的结构体 */
struct imx_pmx_func {
    const char *name;
    const char **groups;
    unsigned num_groups;
};
	struct imx_pin_group *grp;
	/* 涉及到的结构体 */
struct imx_pin_group {
	const char *name;
	unsigned npins;
	unsigned int *pin_ids;
	struct imx_pin *pins;
};


	/* 初始化functions */
	func->name = np->name;		/* 保存functions节点名字 */
	func->num_groups = of_get_child_count(np);	/* 统计子节点数量 */

	......
	for_each_child_of_node(np, child) {
		func->groups[i] = child->name;	/* 将每个groups节点名保存到imx_pmx_func结构体*/
		grp = &info->groups[info->group_index++];	/* 保存每一个groups结构体 */
		imx_pinctrl_parse_groups(child, grp, info, i++);	/* 解析每一个groups组 */
	}
```

### imx_pinctrl_parse_groups 函数分析

```c
	struct imx_pin_group *grp;
	/* 涉及到的结构体 */
struct imx_pin_group {
	const char *name;
	unsigned npins;
	unsigned int *pin_ids;
	struct imx_pin *pins;
};

	list = of_get_property(np, "fsl,pins", &size);	/* 保存每个groups的大小保存到size */
	......
    /* 根据情况解析每个组的引脚信息 */
    for (i = 0; i < grp->npins; i++) {
		if (info->flags & IMX8_USE_SCU)
			imx_pinctrl_parse_pin_scu(info, &grp->pin_ids[i],
				&grp->pins[i], list_p);
		else
			imx_pinctrl_parse_pin_mem(info, &grp->pin_ids[i],
				&grp->pins[i], list_p);
	}

```

### imx_pinctrl_parse_pin_mem 函数分析 

```c
	/* 涉及到的结构体 */
struct imx_pin {
	unsigned int pin;
	union {
		struct imx_pin_memmap pin_memmap;
		struct imx_pin_scu pin_scu;
	} pin_conf;
};

struct imx_pin_memmap {
	unsigned int mux_mode;
	u16 input_reg;
	unsigned int input_val;
	unsigned long config;
};

/**
 * struct imx_pin_reg - describe a pin reg map
 * @mux_reg: mux register offset
 * @conf_reg: config register offset
 */
struct imx_pin_reg {
	s16 mux_reg;
	s16 conf_reg;
};

	/* 保存每个引脚信息 */
	if (info->flags & SHARE_MUX_CONF_REG) {
		conf_reg = mux_reg;
	} else {
		conf_reg = be32_to_cpu(*((*list_p)++));
		if (!conf_reg)
			conf_reg = -1;
	}

	// 相邻引脚的mux_reg之间的偏移量差距为4，根据偏移量算出引脚的id值，引脚的id值pinctrl_imx6ul.c文件中定义
	pin_id = (mux_reg != -1) ? mux_reg / 4 : conf_reg / 4;
	pin_reg = &info->pin_regs[pin_id];
	pin->pin = pin_id;
	*grp_pin_id = pin_id;
	// 复用寄存器地址
	pin_reg->mux_reg = mux_reg;
	// 配置寄存器地址
	pin_reg->conf_reg = conf_reg;
	// 输入寄存器地址
	pin_memmap->input_reg = be32_to_cpu(*((*list_p)++));
	// 配置寄存器的值
	pin_memmap->mux_mode = be32_to_cpu(*((*list_p)++));
	// 输入寄存器的值
	pin_memmap->input_val = be32_to_cpu((*(*list_p)++));

	/* SION bit is in mux register */
	config = be32_to_cpu(*((*list_p)++));
	// IMX_PAD_SION的值为0x40000000 当我们再设备树中将配置寄存器的第30位配置为1时，在驱动源码中会将复用寄存器的SION位配置成	1
	if (config & IMX_PAD_SION)
		pin_memmap->mux_mode |= IOMUXC_CONFIG_SION;
	// 记录配置寄存器的值
	pin_memmap->config = config & ~IMX_PAD_SION;

	dev_dbg(info->dev, "%s: 0x%x 0x%08lx", info->pins[pin_id].name,
			pin_memmap->mux_mode, pin_memmap->config);

	return 0;

```

