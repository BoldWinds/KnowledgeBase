
[添加注释的代码仓库](https://github.com/BoldWinds/AutoDock-Vina-Annotated)

## 基本数据结构与工具
### random
封装了浮点数、有符号/无符号整数、向量的随机生成

### common
- 定义三维向量(vec)和3x3矩阵(mat)结构，提供基本数学运算
- 提供各种类型别名(浮点数、向量容器、文件路径等)
- 实现错误处理机制(内部异常和检查宏)
- 提供数学工具函数(角度标准化、距离计算、数值比较等)
- 实现通用工具函数(字符串转换、容器操作、打印输出等)

### file
封装输入和输出

### matrix
（可能的性能优化点）
#### triangular_matrix_index.h：
提供上三角矩阵的一维数组索引计算功能，用于优化对称矩阵的内存存储
计算过程中可能用到很多对称矩阵，它的存储方式就是用上三角矩阵按列存储成数组。
#### matrix
实现了三种矩阵模板类：**通用矩阵**（支持任意二维矩阵操作）、**三角矩阵**（存储对称矩阵的上三角部分）和**严格三角矩阵**（不含对角线的上三角矩阵）。这些类为AutoDock Vina提供高效的数值计算基础，采用列主序存储和紧凑存储策略优化内存使用。

### convert_substring
支持将字符串指定范围（1基索引）转换为各种数据类型。包含转换函数、空白检查函数和unsigned类型特化处理，应该是用于在读取文件时，将保存的数值字符串转换为特定的数据类型。

### quaternion
这个模块实现了AutoDock Vina中分子旋转操作所需的四元数数学运算，包括四元数与轴角表示的相互转换、旋转矩阵转换、随机旋转生成和旋转增量计算等功能

### int_pow
计算浮点数$x$的$n$次幂：
```cpp
template<unsigned n>
inline fl int_pow(fl x) {
    return int_pow<n-1>(x)*x; 

}

template<>
inline fl int_pow<0>(fl x) {
    return 1;
}
```
因为这部分功能仅在计算一些势能时被调用，`n`只能是常数，比如`int_pow<6>(x)`，编译器在编译时就会把`int_pot<6>(x)`展开为最简单的乘法，相比于递归算法，既减去了运行时的开销，也利于编译器进行优化。

## 原子建模

### atom_constants
定义了AutoDock Vina中三种评分函数(AutoDock4、X-Score、DrugScore-CSD)的原子类型==常量==和==属性参数==，包括原子的范德华半径、势阱深度、氢键参数、溶剂化参数等物理化学性质，并提供了原子类型判断、转换和属性查询的工具函数。

### atom_type
用于统一管理原子在四种不同分类方案(元素、AutoDock4、X-Score、DrugScore-CSD)下的类型信息，提供原子属性查询、类型判断、共价半径计算等功能，并实现了原子对在三角矩阵中的索引计算，是分子建模和能量计算的核心数据结构。

### atom_base
继承`atom_type`，添加电荷信息

### atom
首先定义了原子索引`atom_index`和化学键`bond`。`atom_index`指示了原子的索引编号，`bond`中会保存`atom_index`来表示指向的原子。

`atom`继承自`atom_base`，包含空间坐标和化学键信息。原子本身加上`bond`中保存的化学键信息，就能知道该原子与哪个原子之间用化学键相连

## 能量与评分

### potentials
以`Potential`为基类，定义计算两个原子之间或者两种原子类型之间的势能的虚函数；具体计算势能的类继承`Potential`，包含高斯吸引、排斥、疏水、氢键、静电和溶剂化等主要分子间相互作用。

计算势能时会用到很多参数，比如`offset`和`cutoff`，他们都是人为设置的经验性参数，使理论计算符合实验观察。

### conf_independent

- 首先定义了`conf_independent_inputs`，根据`model`分子模型对象，存储分子的构象无关特征
- 与`Potential`类似，定义了`ConfIndependent`基类，定义了`eval`虚函数，需要把`conf_independent_inputs`和要修正的能量值作为参数
- 实现了继承自`ConfIndependent`的用于扭转角、配体长度、配体数量、重原子疏水原子等相关的构象无关类
### scoring_function
调用potentials.h中已经写好的能量计算方法(`eval`)，结合输入的权重`weights`得到结合能；调用`conf_independent`来应用构象无关项对能量的修正

### precalculate

- 首先定义`precalculate_element`，存储查找表，包括能量快速查找表`fast`和包含能量与导数的查找表`smooth`。`smooth`的能量由`scoring_function`
- `precalculate`和`precalculate_byatom`非常相似，他们本质上都是存储了一个元素类型为`precalculate_element`的上三角矩阵。前者的矩阵内容是根据==原子类型==进行预计算的，后者则是根据一个`model`中的真实原子对进行计算。所以不难发现前者矩阵大小会小一些，精度会差一些；后者的大小则取决于`model`中真实存在的原子数，可能会很大，精度更好。前者适合全局搜索使用，后者适合局部搜索

## 分子建模

### conf
- 定义了AutoDock Vina中分子构象的数据结构，包括扭转键(flv)、刚体、刚体变化、配体、配体变化、柔性残基（受体）
- 定义了操作他们的函数，包括：设置为0、随机生成、检查是否过于接近、随机扰动、根据参考（最佳）构象或随机扰动生成新构象

### tree
- `frame`定义了分子片段的坐标管理，提供局部坐标系转换的方法
- `atom_range`直接定义了原子序列中索引的起始范围
- `atom_frame`继承自`frame`和`atom_range`，能够批量设置坐标、计算力和力矩的总和
分子片段（分子的一部分，往往是一些特定的结构）：
- `rigid_body`继承自`atom_frame`，表示不可变形的分子片段，如果是类似苯环的刚性结构，没有内部扭转角的，就用这个结构定义
- `axis_frame`继承自`atom_frame`，定义绕轴旋转的分子片段，能根据力和力矩设置扭转角的导数
- `segment` 继承自`axis_frame`，定义能相对父框架扭转的片段，存储相对坐标
- `first_segment`继承自`axis_frame`，是整个分子树的根节点
分子树：
- `tree`：递归定义，由当前节点`node`和子树向量构成，全都是`segment`。支持递归的设置整个树的构象；支持计算整个树的梯度
- `heterotree`：定义根节点与分支，根节点类型为`first_segment`或`rigid_body`，而分支类型为`vector<tree<segment>>`。支持配体、残基的构象设置与梯度计算
- `vector_mutable`

`detivative`方法是用来计算能量函数相对于分子构象参数的偏导数

### model
1. **分子系统表示**：定义了完整分子系统的数据结构，包括配体、受体柔性部分、原子坐标等
2. **相互作用管理**：通过`interacting_pairs`管理分子内和分子间的相互作用
3. **能量计算支持**：提供能量评估、梯度计算等接口，支持优化算法
4. **构象操作**：支持构象设置、RMSD计算、几何变换等操作
5. **文件I/O**：提供PDBQT格式的读写功能，保持原始文件上下文


- 初始化`model`：设置每个配体的原子范围、分配化学键、分配原子类型（ad）、初始化相互作用对
- 定义了`ligand`多继承自`flexible_body`和`atom_range`，而`flexible_body`则是一个由`rigid_body`根节点和若干可旋转`segment`分支构成的`heterotree`
- 用`appender`类实现了`model`之间的合并功能，维护正确的原子索引、坐标、相互作用





## parse_pdbqt
- 功能：解析配体/受体并把解析结果以`model`对象的格式返回
- 核心函数：
  - 读文件解析配体：`model parse_ligand_pdbqt_from_file(const std::string& name, atom_type::t atype)`
  - 读字符串解析配体：`model parse_ligand_pdbqt_from_string(const std::string& string_name, atom_type::t atype)`
  - 读文件解析受体：`model parse_receptor_pdbqt(const std::string& rigid_name, const std::string& flex_name, atom_type::t atype)`
- 工作流：
  1. 对于受体的刚性部分，直接按行读原子，调用`parse_pdbqt_atom_string`将相关信息保存到`rigid`结构体；对于配体或者受体的柔性部分，则要创建上下文和`parsing_struct`，调用`parse_pdbqt_aux`配合其他具体的解析函数，结果最终存储在`non_rigid_parsed`
  2. 对于配体或受体的柔性部分，需要进行后处理；受体的刚性部分则不需要
  3. 使用`pdbqt_initializer`结构体的功能，读入`rigid`或`non_rigid_parsed`来构建`model`对象

~~~
刚体部分是不会有ROOT和BRANCH的结构的，因为它没有旋转功能，所以只需要记下来刚体中每一个原子的坐标和原子类型（ad）等相关信息，我们就可以描述这个刚体。但是柔性残基则不同，需要记住固定部分的ROOT以及可以旋转的BRANCH。
~~~

### 解析

#### 受体刚体解析

受体的刚性部分解析就是按行读入，遇到`ATOM`或`HETATM`行就调用`parse_pdbqt_atom_string`将该行解析为一个原子，并将其放入`rigid`结构体中的`atoms`字段。

#### 配体/受体柔性残基解析

首先，配体和受体的柔性残基是不一样的：
- 受体的柔性残基文件中会定义多个柔性残基，它们分别被`BEGIN_RES`和`END_RES`包裹；但是配体相当于只有一个“柔性残基”，所以不会有`BEGIN_RES`和`END_RES`
- 配体的最后一行会有`TORSDOF`关键字代表的扭转自由度信息；受体的残基则不会有

由上面即可知道，对配体或者受体柔性残基中的单个残基的解析方式基本相同，只需要在解析配体时加上对`TORSDOF`的解析即可，所以解析的流程基本是：
1. 对于配体或者受体的每一个柔性残基，都调用`parse_pdbqt_aux`解析
2. 在`parse_pdbqt_aux`中，先调用`parse_pdbqt_root`解析`ROOT`部分，之后按行解析
3. 若开头`BRANCH`则调用`parse_pdbqt_branch_aux`开始解析`BRANCH`
4. 若开头为`TORSDOF`且在解析配体则保存扭转自由度信息
5. 若是解析受体柔性残基且开头`END_RES`则终止；配体解析则到读完流的所有内容才终止

##### parsing_struct
处理柔性残基的核心数据结构为`parsing_struct`，它是一个树形结构，核心定义如下：
```cpp
struct parsing_struct {
    // 树形节点结构
    template<typename T>
    struct node_t {
        sz context_index;           // 在上下文中的索引
        parsed_atom a;              // 节点包含的原子
        std::vector<T> ps;          // 子分支向量（递归结构）
    };
    
    boost::optional<sz> immobile_atom;           // 不可移动原子索引
    boost::optional<atom_reference> axis_begin;  // 旋转轴起始
    boost::optional<atom_reference> axis_end;    // 旋转轴结束
    std::vector<node> atoms;                     // 根节点向量
};
```
##### Root解析
由于ROOT段中都是不可移动的原子，且没有复杂的嵌套结构，所以解析方式类似刚体解析：检测到ROOT之后，对于每一行原子定义，直接调用`parse_pdbqt_atom_string`解析，然后把原子和上下文加入到`parsing_struct`中（调用`parsing_struct::add`）

```cpp
void parse_pdbqt_root_aux(std::istream& in, parsing_struct& p, context& c) {
    std::string str;

    while(std::getline(in, str)) {
        add_context(c, str);

        if(str.empty()) {} // 忽略空行
        else if(starts_with(str, "WARNING")) {} // 忽略WARNING行 - AutoDockTools bug workaround
        else if(starts_with(str, "REMARK")) {} // 忽略REMARK行
        else if(starts_with(str, "ATOM  ") || starts_with(str, "HETATM"))
            p.add(parse_pdbqt_atom_string(str), c);
        else if(starts_with(str, "ENDROOT"))
            return;
        else if(starts_with(str, "MODEL"))
            throw pdbqt_parse_error("Unexpected multi-MODEL tag found in flex residue or ligand PDBQT file. "
                                    "Use \"vina_split\" to split flex residues or ligands in multiple PDBQT files.");
        else 
            throw pdbqt_parse_error("Unknown or inappropriate tag found in flex residue or ligand.", str);
    }
}
```

##### Branch解析
1. 调用`parse_pdbqt_branch_aux`解析`BRANCH`行，首先分析这个分支的父节点原子和该分支与父节点连接的原子
2. 通过索引找到父原子之后，给该父原子节点(`node`)的`ps`字段建立一个新的`parsing_struct`，代表这个子分支
3. 以新建的`parsing_struct`调用`parse_pdbqt_branch`解析这个分支中的原子
4. 解析原子仍然是调用`parse_pdbqt_atom_string`并把它加入到`parsing_struct`的`atoms`中。如果是与父结构相连的节点，还要将其索引放入`immobile_atom`

```cpp
void parse_pdbqt_branch_aux(std::istream& in, const std::string& str, parsing_struct& p, context& c) {
    unsigned first, second;
    parse_two_unsigneds(str, "BRANCH", first, second);
    sz i = 0;

    // 查找起始原子编号对应的原子
    for(; i < p.atoms.size(); ++i)
        if(p.atoms[i].a.number == first) {
            p.atoms[i].ps.push_back(parsing_struct());
            parse_pdbqt_branch(in, p.atoms[i].ps.back(), c, first, second);
            break;
        }

    if(i == p.atoms.size())
        throw pdbqt_parse_error("Atom number " + std::to_string(first) + " is missing in this branch.", str);
}
void parse_pdbqt_branch(std::istream& in, parsing_struct& p, context& c, unsigned from, unsigned to) {
    std::string str;

    while(std::getline(in, str)) {
        add_context(c, str);

        if(str.empty()) {} // 忽略空行
        else if(starts_with(str, "WARNING")) {} // 忽略WARNING行 - AutoDockTools bug workaround
        else if(starts_with(str, "REMARK")) {} // 忽略REMARK行
        else if(starts_with(str, "BRANCH")) parse_pdbqt_branch_aux(in, str, p, c);
        else if(starts_with(str, "ENDBRANCH")) {
            unsigned first, second;
            parse_two_unsigneds(str, "ENDBRANCH", first, second);
            if(first != from || second != to) 
                throw pdbqt_parse_error("Inconsistent branch numbers.");
            if(!p.immobile_atom) 
                throw pdbqt_parse_error("Atom " + boost::lexical_cast<std::string>(to) + " has not been found in this branch.");
            return;
        }
        else if(starts_with(str, "ATOM  ") || starts_with(str, "HETATM")) {
            parsed_atom a = parse_pdbqt_atom_string(str);
            if(a.number == to)
                p.immobile_atom = p.atoms.size();
            p.add(a, c);
        }
        else if(starts_with(str, "MODEL"))
            throw pdbqt_parse_error("Unexpected multi-MODEL tag found in flex residue or ligand PDBQT file. "
                                    "Use \"vina_split\" to split flex residues or ligands in multiple PDBQT files.");
        else 
            throw pdbqt_parse_error("Unknown or inappropriate tag found in flex residue or ligand.", str);
    }
}
```


### 后处理
将配体或受体残基的解析树`parsing_struct`和扭转度信息转换为`non_rigid_parsed`结构：
```cpp
enum distance_type {
    DISTANCE_FIXED,    ///< 固定距离（如共价键）
    DISTANCE_ROTOR,    ///< 转子距离（可旋转键）
    DISTANCE_VARIABLE  ///< 可变距离（非键相互作用）
};
struct non_rigid_parsed {
    vector_mutable<ligand> ligands;  ///< 配体向量
    vector_mutable<residue> flex;    ///< 柔性残基向量
    
    mav atoms;     ///< 可移动原子向量
    atomv inflex;  ///< 不可移动原子向量

    distance_type_matrix atoms_atoms_bonds;   ///< 可移动原子间的移动性矩阵
    matrix<distance_type> atoms_inflex_bonds; ///< 可移动原子与不可移动原子的移动性矩阵
    distance_type_matrix inflex_inflex_bonds; ///< 不可移动原子间的移动性矩阵

    // 合并完整的移动性矩阵
    distance_type_matrix mobility_matrix() const {
        distance_type_matrix tmp(atoms_atoms_bonds);
        tmp.append(atoms_inflex_bonds, inflex_inflex_bonds);
        return tmp;
    }
};
```

#### PS：配体/受体在实际对接中的表示
#### 配体后处理
```cpp
void postprocess_ligand(non_rigid_parsed& nr, parsing_struct& p, context& c, unsigned torsdof) {
    VINA_CHECK(!p.atoms.empty());
    nr.ligands.push_back(ligand(flexible_body(rigid_body(p.atoms[0].a.coords, 0, 0)), torsdof)); // postprocess_branch将分配begin和end
    postprocess_branch(nr, p, c, nr.ligands.back());
    nr_update_matrixes(nr); // FIXME ?
}
```
1. 使用解析树中第一个原子作为配体的根原子，以该根原子的坐标为坐标系原点初始化`ligand`；
2. 对分支进行递归的后处理`postprocess_branch`
3. 更新移动性矩阵

#### 柔性残基后处理
1. 把这个残基的所有不可移动原子（ROOT中的所有原子以及直接与ROOT相连的不可移动原子）插入到不可移动集合，并设置对应的`axis_begin`和`axis_end`
2. 对于每一个需要建立的分支（单个不可移动原子不建立分支），都用不可移动原子的位置初始化`main_branch`
3. 对分支进行递归的后处理`postprocess_branch`
4. 更新矩阵
```cpp
void postprocess_residue(non_rigid_parsed& nr, parsing_struct& p, context& c) {
    // 把这个残基的所有不可移动原子插入到不可移动集合；设置axis_begin和axis_end
    VINA_FOR_IN(i, p.atoms) {
        parsing_struct::node& p_node = p.atoms[i];
        p_node.insert_inflex(nr);
        p_node.insert_immobiles_inflex(nr);
    }
    // 处理每一个分支
    VINA_FOR_IN(i, p.atoms) {
        parsing_struct::node& p_node = p.atoms[i];
        VINA_FOR_IN(j, p_node.ps) {
            parsing_struct& ps = p_node.ps[j];
            // 检查该节点是否需要建立分支
            if(!ps.essentially_empty()) { // 不可移动原子已插入 // FIXME ?!
                // 用main_branch创建residue并放入nr的flex向量中
                nr.flex.push_back(main_branch(first_segment(ps.immobile_atom_coords(), 0, 0, p_node.a.coords)));
                postprocess_branch(nr, ps, c, nr.flex.back());
            }
        }
    }
    nr_update_matrixes(nr); // FIXME ?
    VINA_CHECK(nr.atoms_atoms_bonds.dim() == nr.atoms.size());
    VINA_CHECK(nr.atoms_inflex_bonds.dim_1() == nr.atoms.size());
    VINA_CHECK(nr.atoms_inflex_bonds.dim_2() == nr.inflex.size());
}
```


|                 | 配体      | 柔性残基      |
| --------------- | ------- | --------- |
| ROOT原子          | 可移动集合   | 不可移动集合    |
| 分支immobile_atom | 可移动集合   | 不可移动集合    |
| 分支其他原子          | 可移动集合   | 可移动集合     |
| 分支起始坐标          | 第一个原子坐标 | 不可移动原子的坐标 |


#### 通用的分支后处理

1. 设置当前分支的索引范围
2. 将可移动原子插入`nr.atoms`，计算相对于分支原点的坐标
3. 调整所有距离矩阵的大小以适应新增的可移动原子
4. 键连关系矩阵设置：
    - 建立axis_begin/end原子与当前分支的固定键连
    - 在轴原子间设置旋转键连关系
    - 同一分支内的所有原子间设置为固定距离约束
5. 为每个子分支创建`segment`对象并递归调用自身

```cpp
template<typename B> // B == ligand / residue
void postprocess_branch(non_rigid_parsed& nr, parsing_struct& p, context& c, B& b) {
    // 1. 设置分支节点的原子起始索引
    b.node.begin = nr.atoms.size();

    // 2. 插入可移动原子
    VINA_FOR_IN(i, p.atoms) {
        parsing_struct::node& p_node = p.atoms[i];
        if(p.immobile_atom && i == p.immobile_atom.get()) {}
        else p_node.insert(nr, c, b.node.get_origin());
        p_node.insert_immobiles(nr, c, b.node.get_origin());
    }

    // 设置分支节点的原子结束索引
    b.node.end = nr.atoms.size();

    // 3. 更新键连矩阵
    nr_update_matrixes(nr);

    // 4. 添加轴键连关系
    add_bonds(nr, p.axis_begin, b.node);
    add_bonds(nr, p.axis_end  , b.node);
    set_rotor(nr, p.axis_begin, p.axis_end);

    // 5. 设置同一分支内原子间的固定距离约束
    VINA_RANGE(i, b.node.begin, b.node.end)
        VINA_RANGE(j, i+1, b.node.end)
            nr.atoms_atoms_bonds(i, j) = DISTANCE_FIXED; // FIXME

    // 6. 递归处理子分支
    VINA_FOR_IN(i, p.atoms) {
        parsing_struct::node& p_node = p.atoms[i];
        VINA_FOR_IN(j, p_node.ps) {
            parsing_struct& ps = p_node.ps[j];
            if(!ps.essentially_empty()) { // 不可移动原子已插入 // FIXME ?!
                // 创建子分支段
                b.children.push_back(segment(ps.immobile_atom_coords(), 0, 0, p_node.a.coords, b.node)); // postprocess_branch将分配begin和end
                postprocess_branch(nr, ps, c, b.children.back());
            }
        }
    }
    VINA_CHECK(nr.atoms_atoms_bonds.dim() == nr.atoms.size());
    VINA_CHECK(nr.atoms_inflex_bonds.dim_1() == nr.atoms.size());
    VINA_CHECK(nr.atoms_inflex_bonds.dim_2() == nr.inflex.size());
}
```


### 结果转换

核心数据结构为`pdbqt_initializer`

- 对于刚体，初始化只需要把`rigid.atoms`复制到`model.grid_atoms`即可
- 对于配体/柔性残基则需要下面几步：
	  1. 复制`non_rigid_parsed`中的`ligands`和`flex`到`model`中
	  2. 计算总原子数并分配内存
	  3. 处理可移动原子：将`movable_atom`强制转换成`atom`，坐标使用相对坐标；绝对坐标保存在`m.coords`中
	  4. 处理不可移动原子：坐标使用零向量（相对坐标的原点）；绝对坐标保存在`m.coord`中
	  5. 设置上下文

对`model`设置好数据后，调用`model.initialize(mobility)`即可完成`model`的初始化

总结一下
- 受体初始化：添加`m.grid_atoms`, `m.flex`, `m.atoms`, `m.coords`, `m.minus_forces`, `m.m_num_movable_atoms`, `m.flex_context`
- 配体初始化：添加`m.ligands`, `m.atoms`, `m.coords`, `m.minus_forces`, `m.m_num_movable_atoms`, `m.ligands.front().cont`


## model initialize

1. 调用`ligand::set_range`设置每个配体的原子范围。递归方法，因为`ligand`本身是一个`heterotree<rigid_body>`，通过dfs搜索节点与子树的原子范围
2. `void model::assign_bonds(const distance_type_matrix& mobility)`分配化学键
3. `void model::assign_types()`分配系统原子类型
4. `void model::initialize_pairs(const distance_type_matrix& mobility)`初始化相互作用对

### 分配化学键

1. 构建`beads`数据结构，并把所有原子添加到`beads`中
2. 遍历每个原子`i`：
	1. 设置`i`的共价半径与截断值
	2. 初步筛选阶段，遍历所有珠子
		1. 跳过距离超过截断值的珠子，减少计算
		2. 遍历该珠子内的每个原子`j`：
			1. 确定`i`和`j`的距离类型
			2. 若`i`和`j`是同一个原子，或者`i`和`j`的距离为可变距离（不符合建立化学键的条件），则跳过
			3. 计算`i`和`j`的距离，若其小于初筛选的范围，则将其添加到相关原子向量中
	3.  遍历相关原子向量中的每个原子`j`：
		1. 为了避免重复计算，首先确认原子索引`j>i`
		2. 计算`i`和`j`的合适共价键距离（二者共价半径之和）、距离类型和距离
		3. 进行判断，`i`和`j`的距离需要小于二者的合适共价键距离乘放大因子，并且二者之间没有原子阻挡（没有一个原子`k`，它与`i`和`j`都不可移动且距离更近）
		4. 给两个原子添加键的信息

`beads`结构体将空间上接近的原子聚集在一起，避免所有原子对都进行距离计算：
```cpp
struct beads {
    fl radius_sqr;  ///< 珠子半径的平方
    std::vector<std::pair<vec, szv> > data;  ///< 珠子数据：<中心坐标, 原子索引列表>
    void add(sz index, const vec& coords);
}
```
`add`方法会寻找所有现存珠子，若在其中一个珠子的范围内，就将其加入；否则以该原子的坐标为中心新建珠子
### 分配原子系统

对于每个原子：首先根据AutoDock的原子类型分配基本元素类型；之后根据基本类型和一些键合情况设置XS原子类型
实际上这个函数最终会给`atom_type`中的`el`和`xs`赋值
### 初始化相互作用

有四种相互作用类型：
- **配体内部相互作用**：仅1-4相互作用，存储在`ligand.pairs`
- **柔性残基内部相互作用**：仅1-4相互作用，存储在`other_pairs`
- **柔性残基间相互作用**：存储在`other_pairs`
- **大环闭合相互作用**：排除1-2, 1-3, 1-4相互作用

>在一个分子中，如果两个原子，从一个原子出发，经过3个共价键能到达另一个原子，那么这两个原子的相互作用就被称为**1-4相互作用**

```cpp
struct interacting_pair {
    sz type_pair_index; ///< 原子类型对在预计算数据中的索引但是nver used
    sz a;              ///< 第一个原子的索引
    sz b;              ///< 第二个原子的索引

    interacting_pair(sz type_pair_index_, sz a_, sz b_) : type_pair_index(type_pair_index_), a(a_), b(b_) {}

};
```

初始化相互作用要遍历所有原子`i`：
1. 获取`i`所在的配体和所有能与`i`有1-2/1-3/1-4相互作用的原子向量`bonded_atoms`
2. 遍历所有原子索引比`i`大的原子`j`：
	1. `i`和`j`必须是可变距离（固定距离的原子对）且`j`不在`bonded_atoms`；并且`i`和`j`还不能有闭合碰撞或者不匹配的闭合虚原子
	2. 获取原子类型索引，确保他们有效
	3. 建立`interacting_pair`数据结构并放入对应的向量
```cpp
void model::initialize_pairs(const distance_type_matrix& mobility) {
    VINA_FOR_IN(i, atoms) {
        sz i_lig = find_ligand(i);
        szv bonded_atoms = bonded_to(i, 3);     // BUG?排除了1-4及以下的相互作用

        VINA_RANGE(j, i + 1, atoms.size()) {
            if (mobility(i, j) == DISTANCE_VARIABLE && !has(bonded_atoms, j)) {
                if (is_closure_clash(i, j) || is_unmatched_closure_dummy(i, j)) continue;
                
                // 获取原子类型索引
                sz t1 = atoms[i].get(atom_typing_used());
                sz t2 = atoms[j].get(atom_typing_used());
                sz n  = num_atom_types(atom_typing_used());

                if (t1 < n && t2 < n) { 
                    // 两个原子类型在快速计算表中的索引
                    sz type_pair_index = triangular_matrix_index_permissive(n, t1, t2);
                    interacting_pair ip(type_pair_index, i, j);
                    
                    if (is_glue_pair(i, j)) {
                        glue_pairs.push_back(ip);
                    } else if (i_lig < ligands.size() && find_ligand(j) == i_lig) {
                        ligands[i_lig].pairs.push_back(ip);
                    } else if (!is_atom_in_ligand(i) && !is_atom_in_ligand(j)) {
                        other_pairs.push_back(ip);
                    }
                }
            }
        }
    }
}
```


>有一个疑问，代码注释中提到一些相互作用只计算1-4相互作用，但实际上将1-4及以下的所有相互作用都排除了，不知道是否是bug
## model append

合并主要就是拼接`model`中保存的各种各样向量，并且更改索引确保不同`model`中的原子保持一致：
- 合并相互作用
- 合并坐标
- 合并配体、柔性残基及其上下文
- 合并原子
- 计算新的相互作用




## 受体刚性部分能量计算

有两种方式：一种是预计算，使用Affinity Map保存刚体部分在整个对接盒子内的势能；另一种则是精确计算

不管是cache还是non_cache，他们都派生自下面的结构体`igrid`。
```cpp
struct igrid {
	virtual fl eval(const model& m, fl v) const = 0;
	virtual fl eval_intra(model& m, fl v) const = 0;
	virtual fl eval_deriv(model& m, fl v) const = 0;
};
```
- `cache`和`non_cache`都继承此接口
- `eval`和`eval_intra`的区别在于，后者只考虑受体，不会计算任何配体的原子
### cache
主要功能就是读取map、将计算好的map写入磁盘以及==计算并填充网格==。前两者功能在此不多做介绍，主要介绍如何计算并填充网格。`cache`结构体有三个字段：网格的维度`m_gd`、所有的网格节点`grid`构成的向量`m_grids`和斜率`m_slope`用于惩罚网格边界的截断。

`cache`实现填充网格的函数`populate`，他的算法如下：
1. 对原子类型进行预处理：过滤掉不需要的原子类型，把一些细分的类型合并，得到最终的需要进行计算的原子类型向量`needed`，同时初始化`m_grids`
2. 优化网格维度设置，用其初始化`svz_grid`，计算每个体素可能发生相互作用的原子索引列表
3. 遍历每个体素`(x,y,z)`：
	1. 将索引转换为三维坐标，并获取该坐标附近的原子索引向量
	2. 遍历这些原子`i`：
		1. 判断`i`的原子类型和`i`与距离是否符合条件
		2. 对于`needed`中的所有原子类型`j`，调用`precalculate::eval_fast`计算`i`与`j`的相互作用能量，并累加到`affinities[j]`
	3. 遍历完成后，`needed`中的每个原子类型都计算完了在体素`(x,y,z)`处与所有可能发生相互作用的原子类型的能量之和，将其保存到`m_grids`中该原子类型的网格的对应体素中
```cpp
void cache::populate(const model &m, const precalculate &p, const szv &atom_types_needed) {
	// 原子类型预处理
	szv needed;
	bool got_C_H_already = false;
	bool got_C_P_already = false;
	VINA_FOR_IN(i, atom_types_needed) {
		sz t = atom_types_needed[i];
		switch (t)
		{
			case XS_TYPE_G0:
			case XS_TYPE_G1:
			case XS_TYPE_G2:
			case XS_TYPE_G3:
				continue;
			case XS_TYPE_C_H_CG0:
			case XS_TYPE_C_H_CG1:
			case XS_TYPE_C_H_CG2:
			case XS_TYPE_C_H_CG3:
				if (got_C_H_already) continue;
				t = XS_TYPE_C_H;
				got_C_H_already = true;
				break;
			case XS_TYPE_C_P_CG0:
			case XS_TYPE_C_P_CG1:
			case XS_TYPE_C_P_CG2:
			case XS_TYPE_C_P_CG3:
				if (got_C_P_already) continue;
				t = XS_TYPE_C_P;
				got_C_P_already = true;
				break;
		}
		if(!m_grids[t].initialized()) {
			needed.push_back(t);
			m_grids[t].init(m_gd);
		}
	}
	if(needed.empty())
		return;
	
	// 初始化数据结构
	flv affinities(needed.size());
	sz nat = num_atom_types(atom_type::XS);
	grid& g = m_grids[needed.front()];		// 仅用来确定维度
	const fl cutoff_sqr = p.cutoff_sqr();

	// 初始化szv_grid
	grid_dims gd_reduced = szv_grid_dims(m_gd);
	szv_grid ig(m, gd_reduced, cutoff_sqr);

	// 能量计算
	VINA_FOR(x, g.m_data.dim0()) {
		VINA_FOR(y, g.m_data.dim1()) {
			VINA_FOR(z, g.m_data.dim2()) {
				// 对于(x,y,z)体素
				std::fill(affinities.begin(), affinities.end(), 0);
				// 把三维坐标转换为索引
				vec probe_coords; 
				probe_coords = g.index_to_argument(x, y, z);
				// 获取该坐标附近的能产生相互作用的原子索引
				const szv& possibilities = ig.possibilities(probe_coords);
				VINA_FOR_IN(possibilities_i, possibilities) {
					const sz i = possibilities[possibilities_i];
					const atom& a = m.grid_atoms[i];
					const sz t1 = a.get(atom_type::XS);
					// 原子类型不符合就跳过
					if(t1 >= nat) continue;
					// 距离超出阶段范围也跳过
					const fl r2 = vec_distance_sqr(a.coords, probe_coords);
					if(r2 <= cutoff_sqr) {
						VINA_FOR_IN(j, needed) {
							const sz t2 = needed[j];
							assert(t2 < nat);
							const sz type_pair_index = triangular_matrix_index_permissive(nat, t1, t2);
							affinities[j] += p.eval_fast(type_pair_index, r2);
						}
					}
				}
				// 把计算出的亲和力存入网格
				VINA_FOR_IN(j, needed) {
					sz t = needed[j];
					assert(t < nat);
					m_grids[t].m_data(x, y, z) = affinities[j];
				}
			}
		}
	}
}
```

>为什么在遍历体素时，先把索引转换为坐标，又把坐标转换为索引？
>因为svz_grid中用到的网格维度设置是优化后的
#### grid

```cpp
class grid {
public:
    vec m_init;        ///< 网格起始坐标 (网格空间的原点)
    vec m_range;       ///< 网格范围 (每个维度的总长度)
    vec m_factor_inv;  ///< 缩放因子的倒数 (用于从网格索引转换为实际坐标)
    array3d<fl> m_data; ///< 三维数组，存储该原子类型对应的预计算能量
private:
    vec m_factor;          ///< 转换因子 (用于从实际坐标转换为网格索引)
    vec m_dim_fl_minus_1;  ///< 网格维度减1 (用于边界检查)
	/**
     * @brief 网格能量计算的核心实现函数
     */
	fl evaluate_aux(const vec& location, fl slope, fl v, vec* deriv) const; 
};
```

核心函数是`evaluate_aux`：
1. 对于输入的坐标，转换为带小数部分的网格索引
2. 进行边界检查与越界的处理；如果没有越界则将索引的小数部分和整数部分分别存储
3. 三线性插值计算能量和导数
```cpp
fl grid::evaluate_aux(const vec& location, fl slope, fl v, vec* deriv) const {
	vec miss(0, 0, 0);
	boost::array<int, 3> region;
	boost::array<sz, 3> a;	// 网格索引的整数部分

	// 含小数部分的网格索引，会在处理后减去整数部分仅保留小数部分
	vec s  = elementwise_product(location - m_init, m_factor); 

	// 边界检查与越界处理
	VINA_FOR(i, 3) {
		if(s[i] < 0) {	// 下界外
			miss[i] = -s[i];
			region[i] = -1;
			a[i] = 0; 
			s[i] = 0;
		}       
		else if(s[i] >= m_dim_fl_minus_1[i]) {	// 上界外
			miss[i] = s[i] - m_dim_fl_minus_1[i];
			region[i] = 1;
			assert(m_data.dim(i) >= 2);
			a[i] = m_data.dim(i) -  2; 
			s[i] = 1;
		}
		else {	// 在网格中的情况
			region[i] = 0;
			a[i] = sz(s[i]);	// 保留索引的整数部分
			s[i] -= a[i];		// 保留索引的小数部分
		}
		assert(s[i] >= 0);
		assert(s[i] <= 1);
		assert(a[i] >= 0);
		assert(a[i]+1 < m_data.dim(i));
	}
	const fl penalty = slope * (miss * m_factor_inv);
	assert(penalty > -epsilon_fl);

	// 三线性插值
	const sz x0 = a[0];
	const sz y0 = a[1];
	const sz z0 = a[2];

	const sz x1 = x0+1;
	const sz y1 = y0+1;
	const sz z1 = z0+1;


	const fl f000 = m_data(x0, y0, z0);
	const fl f100 = m_data(x1, y0, z0);
	const fl f010 = m_data(x0, y1, z0);
	const fl f110 = m_data(x1, y1, z0);
	const fl f001 = m_data(x0, y0, z1);
	const fl f101 = m_data(x1, y0, z1);
	const fl f011 = m_data(x0, y1, z1);
	const fl f111 = m_data(x1, y1, z1);

	const fl x = s[0];
	const fl y = s[1];
	const fl z = s[2];

	const fl mx = 1-x;
	const fl my = 1-y;
	const fl mz = 1-z;

	fl f = 
		f000 *  mx * my * mz  +
		f100 *   x * my * mz  +
		f010 *  mx *  y * mz  + 
		f110 *   x *  y * mz  +
		f001 *  mx * my *  z  +
		f101 *   x * my *  z  +
		f011 *  mx *  y *  z  +
		f111 *   x *  y *  z  ;

	// 梯度计算
	if(deriv) { // valid pointer
		const fl x_g = 
			f000 * (-1)* my * mz  +
			f100 *   1 * my * mz  +
			f010 * (-1)*  y * mz  + 
			f110 *   1 *  y * mz  +
			f001 * (-1)* my *  z  +
			f101 *   1 * my *  z  +
			f011 * (-1)*  y *  z  +
			f111 *   1 *  y *  z  ;


		const fl y_g = 
			f000 *  mx *(-1)* mz  +
			f100 *   x *(-1)* mz  +
			f010 *  mx *  1 * mz  + 
			f110 *   x *  1 * mz  +
			f001 *  mx *(-1)*  z  +
			f101 *   x *(-1)*  z  +
			f011 *  mx *  1 *  z  +
			f111 *   x *  1 *  z  ;


		const fl z_g =  
			f000 *  mx * my *(-1) +
			f100 *   x * my *(-1) +
			f010 *  mx *  y *(-1) + 
			f110 *   x *  y *(-1) +
			f001 *  mx * my *  1  +
			f101 *   x * my *  1  +
			f011 *  mx *  y *  1  +
			f111 *   x *  y *  1  ;

		vec gradient(x_g, y_g, z_g);
		curl(f, gradient, v);
		vec gradient_everywhere;

		VINA_FOR(i, 3) {
			gradient_everywhere[i] = ((region[i] == 0) ? gradient[i] : 0);	// 越界则为0
			(*deriv)[i] = m_factor[i] * gradient_everywhere[i] + slope * region[i];	// 如果越界，导数的方向会指回界内，便于收敛
		}

		return f + penalty;
	}
	else {
		curl(f, v);
		return f + penalty;
	}
} 
```

### non_cache

```cpp
struct non_cache : public igrid {
    non_cache() {}
	non_cache(const model& m, const grid_dims& gd_, const precalculate* p_, fl slope_);
	virtual fl eval      (const model& m, fl v) const;
	virtual fl eval_intra(      model& m, fl v) const;
	virtual fl eval_deriv(      model& m, fl v) const; 
	bool within(const model& m, fl margin = 0.0001) const;
	fl slope;
private:
	szv_grid sgrid;
	grid_dims gd;
	const precalculate* p;
};
```

直接计算能量，eval的的实现如下：
1. 初始化总能量值、截断距离和原子类型数
2. 遍历所有可移动原子`i`：
	1. 处理原子类型，进行过滤/合并
	2. 边界情况处理，把越界原子投影到网格边界上，并且设置越界惩罚
	3. 查找可能相互作用原子，并遍历`j`：
3. 返回总能量值
```cpp
fl non_cache::eval      (const model& m, fl v) const { // clean up
	fl e = 0;
	const fl cutoff_sqr = p->cutoff_sqr();

	sz n = num_atom_types(atom_type::XS);

	VINA_FOR(i, m.num_movable_atoms()) {
		fl this_e = 0;
		fl out_of_bounds_penalty = 0;
		const atom& a = m.atoms[i];
		sz t1 = a.get(atom_type::XS);
		if(t1 >= n) continue;
        switch (t1)
        {
			case XS_TYPE_G0:
			case XS_TYPE_G1:
			case XS_TYPE_G2:
			case XS_TYPE_G3:
				continue;
			case XS_TYPE_C_H_CG0:
			case XS_TYPE_C_H_CG1:
			case XS_TYPE_C_H_CG2:
			case XS_TYPE_C_H_CG3:
				t1 = XS_TYPE_C_H;
				break;
			case XS_TYPE_C_P_CG0:
			case XS_TYPE_C_P_CG1:
			case XS_TYPE_C_P_CG2:
			case XS_TYPE_C_P_CG3:
				t1 = XS_TYPE_C_P;
				break;
		}

		// 边界处理
		const vec& a_coords = m.coords[i];
		vec adjusted_a_coords; adjusted_a_coords = a_coords;
		VINA_FOR_IN(j, gd) {
			if(gd[j].n_voxels > 0) {
				if     (a_coords[j] < gd[j].begin) { adjusted_a_coords[j] = gd[j].begin; out_of_bounds_penalty += std::abs(a_coords[j] - gd[j].begin); }
				else if(a_coords[j] > gd[j].end  ) { adjusted_a_coords[j] = gd[j]  .end; out_of_bounds_penalty += std::abs(a_coords[j] - gd[j]  .end); }
			}
		}
		out_of_bounds_penalty *= slope;

		// 可能与i产生相互作用的原子
		const szv& possibilities = sgrid.possibilities(adjusted_a_coords);
		VINA_FOR_IN(possibilities_j, possibilities) {
			const sz j = possibilities[possibilities_j];
			const atom& b = m.grid_atoms[j];
			sz t2 = b.get(atom_type::XS);
			if(t2 >= n) continue;
			vec r_ba; r_ba = adjusted_a_coords - b.coords; // FIXME why b-a and not a-b ?
			fl r2 = sqr(r_ba);
			if(r2 < cutoff_sqr) {
				sz type_pair_index = get_type_pair_index(atom_type::XS, a, b);
				this_e +=  p->eval_fast(type_pair_index, r2);
			}
		}
		curl(this_e, v);
		e += this_e + out_of_bounds_penalty;
	}
	return e;
}
```

`non_cache`计算的精确性主要体现在于：
1. 直接使用两个原子的距离查表，而非用插值
2. 使用距离导数计算更精确
3. 边界处理更准确




# Prompt
用户提供的代码是AutoDock Vina源代码的一部分，AutoDock Vina是一个用于药物筛选的开源软件。你是一个资深程序员，请按以下步骤处理用户提供的代码：
1. **完整理解**：逐行分析代码逻辑，确保完全掌握其功能和技术细节
2. **注释添加**：为所有关键元素添加符合Doxygen规范的**中文注释**，包括：
	- 文件头部注释（仅需提供文件名、简要描述，版权和作者在代码中都已给出了，将它们合并到头部注释即可）
	- 函数/类注释（功能描述、参数说明、返回值）
	- 复杂逻辑段落的行内注释
3. **功能总结**：分析总结该代码在Autodock Vina分子对接中的作用

## 处理原则
1. 注释必须**准确反映代码行为**，不添加虚构功能
2. 对复杂算法使用`@note`解释实现原理
3. 保持原始代码结构不变，缩进使用tab