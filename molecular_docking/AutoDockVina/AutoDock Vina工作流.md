1. 检查参数，初始化`Vina`类
2. 解析受体：`v.set_receptor`
3. 解析配体：`v.set_ligand_from_file`
4. 分别进行`model`的初始化
5. 将得到的多个`model`合并
6. 加载配体`map`：`v.load_maps`；若没有则计算`map`：`v.compute_vina_maps`
7. 根据设置调用具体的计算，默认为`v.global_search`+`v.write_poses`

---
# 初始化

```cpp
Vina(const std::string &sf_name="vina", int cpu=0, int seed=0, int verbosity=1, bool no_refine=false, std::function<void(double)>* progress_callback = NULL)
```

- `sf_name`: 评分函数名称，可以为"vina"/"vinardo"/"ad4"，会自动设置对应的默认权重
- `cpu`：使用的cpu线程数，默认使用全部
- `seed`：用于生成随机数的种子
- `verbosity`：日志级别
- `no_refine`：是否启用精细计算，默认false为启用
- `progress_callback`：回调函数，目前的代码中全部都是`NULL`

---
# 解析受体/配体

解析配体/受体并把解析结果以`model`对象的格式返回，有三个核心函数：
- 读文件解析配体：`model parse_ligand_pdbqt_from_file(const std::string& name, atom_type::t atype)`
- 读字符串解析配体：`model parse_ligand_pdbqt_from_string(const std::string& string_name, atom_type::t atype)`
- 读文件解析受体：`model parse_receptor_pdbqt(const std::string& rigid_name, const std::string& flex_name, atom_type::t atype)`

不管是对于受体还是配体的解析，遵循以下的流程：
1. 解析输入到一个数据结构中`rigid`/`parsing_struct`
2. （把`parsing_struct`转换为`non_rigid_parsed`）
3. 用该数据结构初始化一个`model`
## pdbqt文件

### 受体刚性示例

刚体部分往往都是认为固定不动的，所以其pdbqt文件含有信息的行就只有一个个的`ATOM`行
```
ATOM      1  N   ALA A   1      20.154  15.234   8.123  0.00  0.00    -0.347 N 
ATOM      2  CA  ALA A   1      19.876  16.543   8.754  0.00  0.00     0.221 C 
ATOM      3  C   ALA A   1      18.456  16.612   9.295  0.00  0.00     0.763 C 
ATOM      4  O   ALA A   1      17.512  16.012   8.876  0.00  0.00    -0.801 O 
ATOM      5  CB  ALA A   1      20.087  17.701   7.832  0.00  0.00     0.037 C 
ATOM      6  N   VAL A   2      18.321  17.432  10.341  0.00  0.00    -0.347 N 
ATOM      7  CA  VAL A   2      17.032  17.634  11.012  0.00  0.00     0.221 C 
ATOM      8  C   VAL A   2      16.987  16.743  12.245  0.00  0.00     0.763 C 
ATOM      9  O   VAL A   2      17.912  16.543  12.987  0.00  0.00    -0.801 O 
ATOM     10  CB  VAL A   2      16.832  19.076  11.432  0.00  0.00     0.012 C 
ATOM     11  CG1 VAL A   2      15.543  19.234  12.187  0.00  0.00     0.024 C 
ATOM     12  CG2 VAL A   2      16.789  20.043  10.287  0.00  0.00     0.024 C 
TER
```

### 配体示例

- 用`ROOT`和`BRANCH`来表示树状的结构
- 最后的`TORSDOF`是该配体的扭转度

```
ROOT
ATOM      1  C1  LIG A   1      15.432  18.234  16.123  0.00  0.00     0.170 A 
ATOM      2  C2  LIG A   1      16.765  17.876  16.432  0.00  0.00     0.050 A 
ATOM      3  C3  LIG A   1      17.234  16.654  16.876  0.00  0.00     0.007 A 
ATOM      4  C4  LIG A   1      16.432  15.543  17.123  0.00  0.00     0.000 A 
ATOM      5  C5  LIG A   1      15.123  15.876  16.876  0.00  0.00     0.007 A 
ATOM      6  C6  LIG A   1      14.654  17.123  16.432  0.00  0.00     0.050 A 
ENDROOT
BRANCH   2   7
ATOM      7  C7  LIG A   1      17.543  19.087  16.123  0.00  0.00     0.308 C 
BRANCH   7   8
ATOM      8  C8  LIG A   1      18.876  19.234  16.876  0.00  0.00     0.253 C 
ATOM      9  O9  LIG A   1      19.123  18.765  17.987  0.00  0.00    -0.270 OA
ENDBRANCH   7   8
ENDBRANCH   2   7
BRANCH   1  10
ATOM     10  O10 LIG A   1      14.876  19.432  15.765  0.00  0.00    -0.356 OA
ATOM     11  H10 LIG A   1      14.123  19.654  16.234  0.00  0.00     0.218 HD
ENDBRANCH   1  10
TORSDOF 4
```

### 受体柔性残基示例

可以认为是由一个个的`BEGIN_RES`和`END_RES`包裹的"配体"，只不过不会有`TORSDOF`行

## 受体刚性解析

解析结果用`rigid`存储，非常简单的结构体：
```cpp
struct rigid {
    atomv atoms; ///< 刚性原子向量
};
```

遇到`ATOM`或`HETATM`行就调用`parse_pdbqt_atom_string`将该行解析为一个原子，并将其放入`rigid`结构体中的`atoms`字段

`parse_pdbqt_atom_string`的作用就是将参数的字符串解析出原子索引、三维坐标、电荷、原子类型，并创建`parsed_atom`对象存入这些信息

### PS: AutoDock Vina的原子表示

![Atom类图](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506172149777.png)

- `atom_type`中保存原子在不同分类方案下的类型信息，还提供了获取该类型原子共价半径等与原子类型相关的方法
- `atom_base`：添加了电荷信息
- `atom`：添加了坐标与该原子所有的化学键
- `parsed_atom`：添加了`number`字段用于保存解析时的pdbqt的原子序号，解析时的默认原子，但最终只用于表示不可移动的原子
- `movable_atom`：添加了该原子相对刚体远点的坐标，一般是从`parsed_atom`强制转换过来用于描述可移动原子


## 配体/柔性残基解析

由于配体和柔性残基几乎可以认为是元素和向量的关系，所以对柔性残基的解析可以在配体解析上扩展进行，基本的解析流程如下：
1. 对于配体或者受体的每一个柔性残基，都分配一个`parsing_struct`，调用`parse_pdbqt_aux`解析
2. 先调用`parse_pdbqt_root`解析`ROOT`部分，之后按行解析
3. 若开头`BRANCH`则调用`parse_pdbqt_branch_aux`开始解析`BRANCH`
4. 若开头为`TORSDOF`且在解析配体则保存扭转自由度信息
5. 若是解析受体柔性残基且开头`END_RES`则终止；配体解析则到读完流的所有内容才终止

### `parsing_struct`
处理配体/柔性残基的核心数据结构为`parsing_struct`，它是一个树形结构，核心定义如下：
```cpp
struct parsing_struct {
    // 树形节点结构
    template<typename T>  // T 往往是parsing_struct
    struct node_t {
        sz context_index;           // 在上下文中的索引
        parsed_atom a;              // 节点包含的原子
        std::vector<T> ps;          // 子分支向量（递归结构）
    };
	typedef node_t<parsing_struct> node;
    
    boost::optional<sz> immobile_atom;           // 不可移动原子索引
    boost::optional<atom_reference> axis_begin;  // 旋转轴起始原子引用
    boost::optional<atom_reference> axis_end;    // 旋转轴结束原子引用
    std::vector<node> atoms;                     // 根节点向量
};
```

它形成一个树的结构：`atoms`中保存根部的所有节点，每个节点都包含一个原子和若干个子树；

### ROOT解析

由于ROOT段中没有复杂的嵌套结构，所以解析方式类似刚体解析：检测到ROOT之后，对于每一行原子定义，直接调用`parse_pdbqt_atom_string`解析，然后把原子和上下文加入到`parsing_struct`中（调用`parsing_struct::add`）

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
### BRANCH解析

1. 调用`parse_pdbqt_branch_aux`解析`BRANCH`行，首先分析这个分支的父节点原子和该分支与父节点连接的原子
2. 通过索引找到父原子之后，给该父原子节点(`node`)的`ps`字段建立一个新的`parsing_struct`，代表这个子分支
3. 以新建的`parsing_struct`调用`parse_pdbqt_branch`解析这个分支中的原子
4. 解析原子仍然是调用`parse_pdbqt_atom_string`并把它加入到`parsing_struct`的`atoms`中。如果是与父结构相连的节点，还要将其索引放入`immobile_atom`；`parsing_struct`中的`axis_begin`和`axis_end`在后处理中才会被设置

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
            // 将本分支与父节点连接的原子设为不可移动
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

## 后处理
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

### PS：配体/受体在实际对接中的表示
这部分内容在`tree.h`中定义：
![配受体类图](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506180016705.png)

- `frame`：定义坐标系的原点位置和旋转方向
- `atom_range`：只有`begin`和`end`字段，用起始和截止索引规定原子范围
- `atom_frame`：派生自`frame`和`atom_range`，集合了一组在同一个坐标系内的原子
- `rigid_body`：没有添加新的字段，支持用刚体构象设置`rigid_body`
- `axis_frame`：添加旋转轴单位向量，他用与轴根位置与坐标系原点的差确定
- `first_segment`：没有添加新的字段，作为tree的根节点
- `segment`：添加了相对父节点的相对旋转和相对坐标

- `rigid_body`有完整的6个自由度（平移和旋转），对应配体的根部分；
- `first_segment`只能绕固定轴旋转，对应受体的柔性残基必须固定在特定的连接点
- `segment`是相对父节点的旋转分支，对应配体/柔性残基的分支

#### tree & heterotree
```cpp
template<typename T> // T == segment / first_segmeent
struct tree {
	T node;	 ///< 当前节点
	std::vector< tree<T> > children; ///< 子树向量
};

typedef tree<segment> branch;           ///< 分支类型定义
typedef std::vector<branch> branches;   ///< 分支向量类型定义

template<typename Node> // Node == rigid_body / first_segment
struct heterotree {
	Node node;			///< 根节点
	branches children;	///< 子分支
	heterotree(const Node& node_) : node(node_) {}
};
```

#### 配体与残基
```cpp
typedef heterotree<rigid_body> flexible_body;
typedef heterotree<first_segment> main_branch;

// ligand(flexible_body(rigid_body(p.atoms[0].a.coords, 0, 0)), torsdof)
struct ligand : public flexible_body, atom_range {
    unsigned degrees_of_freedom; ///< 自由度数量
    interacting_pairs pairs;     ///< 配体内部的相互作用对
    context cont;               ///< 配体的PDBQT文件上下文
	
    ligand(const flexible_body& f, unsigned degrees_of_freedom_) : flexible_body(f), atom_range(0, 0), degrees_of_freedom(degrees_of_freedom_) {}
    
    void set_range();
};

// main_branch(first_segment(ps.immobile_atom_coords(), 0, 0, p_node.a.coords))
struct residue : public main_branch {
    residue(const main_branch& m) : main_branch(m) {}
};
```

### 配体后处理
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

### 柔性残基后处理
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

### 通用的分支后处理

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


## 将解析结果转换为对接模型`model`

核心数据结构为`pdbqt_initializer`

- 对于刚体，初始化只需要把`rigid.atoms`复制到`model.grid_atoms`即可
- 对于配体/柔性残基则需要下面几步：
	  1. 赋值`non_rigid_parsed`中的`ligands`和`flex`到`model`中
	  2. 计算总原子数并分配内存
	  3. 处理可移动原子：将`movable_atom`强制转换成`atom`，坐标使用相对坐标；绝对坐标保存在`m.coords`中
	  4. 处理不可移动原子：坐标使用零向量（相对坐标的原点）；绝对坐标保存在`m.coord`中
	  5. 设置上下文

```cpp
void initialize_from_nrp(const non_rigid_parsed& nrp, const context& c, bool is_ligand) { // static really
    VINA_CHECK(m.ligands.empty());
    VINA_CHECK(m.flex   .empty());
    // 复制配体和柔性残基数据
    m.ligands = nrp.ligands;
    m.flex    = nrp.flex;
    VINA_CHECK(m.atoms.empty());
    // 计算总原子数并预分配内存
    sz n = nrp.atoms.size() + nrp.inflex.size();
    m.atoms.reserve(n);
    m.coords.reserve(n);
    // 处理可移动原子：设置相对坐标用于旋转计算
    VINA_FOR_IN(i, nrp.atoms) {
        const movable_atom& a = nrp.atoms[i];
        atom b = static_cast<atom>(a);
        b.coords = a.relative_coords;  // 使用相对坐标
        m.atoms.push_back(b);
        m.coords.push_back(a.coords);  // 保存实际坐标
    }
    // 处理不可移动原子：坐标设为零向量避免混淆
    VINA_FOR_IN(i, nrp.inflex) {
        const atom& a = nrp.inflex[i];
        atom b = a;
        b.coords = zero_vec; // 避免混淆，这些坐标不会被访问
        m.atoms.push_back(b);
        m.coords.push_back(a.coords);
    }
    VINA_CHECK(m.coords.size() == n);
    // 初始化力计算相关数据结构
    m.minus_forces = m.coords;
    m.m_num_movable_atoms = nrp.atoms.size();
    // 根据类型设置上下文信息
    if(is_ligand) {
        VINA_CHECK(m.ligands.size() == 1);
        m.ligands.front().cont = c;  // 配体使用前端上下文
    }
    else
        m.flex_context = c;          // 柔性残基使用flex上下文
}
```

- 受体初始化：添加`m.grid_atoms`, `m.flex`, `m.atoms`, `m.coords`, `m.minus_forces`, `m.m_num_movable_atoms`, `m.flex_context`
- 配体初始化：添加`m.ligands`, `m.atoms`, `m.coords`, `m.minus_forces`, `m.m_num_movable_atoms`, `m.ligands.front().cont`

# 对接模型的初始化

1. 调用`ligand::set_range`设置每个配体的原子范围。递归方法，因为`ligand`本身是一个`heterotree<rigid_body>`，通过dfs搜索节点与子树的原子范围
2. `void model::assign_bonds(const distance_type_matrix& mobility)`分配化学键
3. `void model::assign_types()`分配系统原子类型
4. `void model::initialize_pairs(const distance_type_matrix& mobility)`初始化相互作用对
```cpp
void model::initialize(const distance_type_matrix& mobility) {
    VINA_FOR_IN(i, ligands)
        ligands[i].set_range();  // 设置每个配体的原子范围
    assign_bonds(mobility);      // 分配化学键
    assign_types();             // 分配原子类型
    initialize_pairs(mobility); // 初始化相互作用对
}
```
## 分配化学键

`beads`结构体将空间上接近的原子聚集在一起，避免所有原子对都进行距离计算：
```cpp
struct beads {
    fl radius_sqr;  ///< 珠子半径的平方
    std::vector<std::pair<vec, szv> > data;  ///< 珠子数据：<中心坐标, 原子索引列表>
    void add(sz index, const vec& coords);
}
```
`add`方法会寻找所有现存珠子，若在其中一个珠子的范围内，就将其加入；否则以该原子的坐标为中心新建珠子

分配算法：
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
```cpp
void model::assign_bonds(const distance_type_matrix& mobility) {
    const fl bond_length_allowance_factor = 1.1;  // 给与键长相比默认值（两个原子共价半径之和）可能变化的范围
    sz n = grid_atoms.size() + atoms.size();  // 总原子数

    // 构建珠子数据结构并将所有原子添加到珠子中
    const fl bead_radius = 15;
    beads beads_instance(n, sqr(bead_radius));
    VINA_FOR(i, n) {
        atom_index i_atom_index = sz_to_atom_index(i);
        beads_instance.add(i, atom_coords(i_atom_index));
    }
    
    // 为每个原子分配键
	VINA_FOR(i, n) {
		atom_index i_atom_index = sz_to_atom_index(i);
		const vec& i_atom_coords = atom_coords(i_atom_index);
		atom& i_atom = get_atom(i_atom_index);

        // 设置共价半径与截断值
		const fl max_covalent_r = max_covalent_radius(); // FIXME mv to atom_constants
		fl i_atom_covalent_radius = max_covalent_r;
		if(i_atom.ad < AD_TYPE_SIZE)
			i_atom_covalent_radius = ad_type_property(i_atom.ad).covalent_radius;
		const fl bead_cutoff_sqr = sqr(bead_radius + bond_length_allowance_factor * (i_atom_covalent_radius + max_covalent_r));
        szv relevant_atoms;
        
        // 遍历所有珠子
		VINA_FOR_IN(b, beads_instance.data) {
            // 跳过距离过远的珠子
			if(vec_distance_sqr(beads_instance.data[b].first, i_atom_coords) > bead_cutoff_sqr) continue;
            
            // 遍历珠子内的原子
			const szv& bead_elements = beads_instance.data[b].second;
			VINA_FOR_IN(bead_elements_i, bead_elements) {
				sz j = bead_elements[bead_elements_i];
				atom_index j_atom_index = sz_to_atom_index(j);
				atom& j_atom = get_atom(j_atom_index);

                // 确定共价键长度和距离类型
				const fl bond_length = i_atom.optimal_covalent_bond_length(j_atom);
				distance_type dt = distance_type_between(mobility, i_atom_index, j_atom_index);
                
                // 若距离可变则不太可能建立化学键
				if(dt != DISTANCE_VARIABLE && i != j) {
					fl r2 = distance_sqr_between(i_atom_index, j_atom_index);
                    // 基于共价半径判断是否在键合范围内
					if(r2 < sqr(bond_length_allowance_factor * (i_atom_covalent_radius + max_covalent_r)))
						relevant_atoms.push_back(j);
				}
			}
		}
        
        // 在相关原子中寻找真正的键合原子
		VINA_FOR_IN(relevant_atoms_i, relevant_atoms) {
			sz j = relevant_atoms[relevant_atoms_i];
            if(j <= i) continue; // 避免重复处理（每对原子只处理一次）
            
			atom_index j_atom_index = sz_to_atom_index(j);
			atom& j_atom = get_atom(j_atom_index);
			const fl bond_length = i_atom.optimal_covalent_bond_length(j_atom);
			distance_type dt = distance_type_between(mobility, i_atom_index, j_atom_index);
			fl r2 = distance_sqr_between(i_atom_index, j_atom_index);
            
            // 最终键分配判断：距离合适且无中间原子阻挡
			if(r2 < sqr(bond_length_allowance_factor * bond_length) && 
               !atom_exists_between(mobility, i_atom_index, j_atom_index, relevant_atoms)) {
                bool rotatable = (dt == DISTANCE_ROTOR);  // 是否为可旋转键
				fl length = std::sqrt(r2);
                
                // 为两个原子都添加键信息
				i_atom.bonds.push_back(bond(j_atom_index, length, rotatable));
				j_atom.bonds.push_back(bond(i_atom_index, length, rotatable));
			}
		}
	}
}
```


## 分配原子系统
对于每个原子：首先根据AutoDock的原子类型分配基本元素类型；之后根据基本类型和一些键合情况设置XS原子类型
实际上就是给每个原子的`atom_type`中的`el`和`xs`赋值
## 初始化相互作用

有四种相互作用类型：
- **配体内部相互作用**：仅1-4相互作用，存储在`ligand.pairs`
- **柔性残基内部相互作用**：仅1-4相互作用，存储在`other_pairs`
- **柔性残基间相互作用**：存储在`other_pairs`
- **大环闭合相互作用**：排除1-2, 1-3, 1-4相互作用

>在一个分子中，如果两个原子，从一个原子出发，经过3个共价键能到达另一个原子，那么这两个原子的相互作用就被称为**1-4相互作用**

初始化相互作用要遍历所有原子`i`：
1. 获取`i`所在的配体和所有能与`i`有1-2/1-3/1-4相互作用的原子向量`bonded_atoms`
2. 遍历所有原子索引比`i`大的原子`j`：
	1. `i`和`j`必须是可变距离（固定距离的原子对）且`j`不在`bonded_atoms`；并且`i`和`j`还不能有闭合碰撞或者不匹配的闭合虚原子
	2. 获取原子类型索引，确保他们有效
	3. 建立`interacting_pair`数据结构并放入对应的向量

```cpp
struct interacting_pair {
    sz type_pair_index; ///< 原子类型对在预计算数据中的索引
    sz a;              ///< 第一个原子的索引
    sz b;              ///< 第二个原子的索引

    interacting_pair(sz type_pair_index_, sz a_, sz b_) : type_pair_index(type_pair_index_), a(a_), b(b_) {}

};
```

>有一个疑问，代码注释中提到一些相互作用只计算1-4相互作用，但实际上将1-4及以下的所有相互作用都排除了，不知道是否是bug

# 对接模型的合并

合并主要就是拼接`model`中保存的各种各样向量，并且更改索引确保不同`model`中的原子保持一致：
- 合并相互作用
- 合并坐标
- 合并配体、柔性残基及其上下文
- 合并原子
- 计算新的相互作用

# 能量网格的设置与计算

由于刚体部分认为它是不动的，所以可以提前计算出任何一个可以移动的原子在对接盒子的任何一个位置（实际上受到分辨率的影响）与所有刚体原子的相互作用能量之和 

这部分功能在`Vina::compute_vina_maps(double center_x, double center_y, double center_z, double size_x, double size_y, double size_z, double granularity, bool force_even_voxels)`中完成,它的大致过程如下:
1. 确定网格的中心位置以及尺寸，并初始化网格的三维维度和体素数量
2. 保存所有可移动原子的原子类型`atom_types`（受体柔性残基和配体）
3. 初始化原子类型的预计算`precalculated_sf`
4. 创建`cache`对象，并调用`populate(m_model, precalculated_sf, atom_types)`填充`atom_types`中所有原子类型所对应的网格对象`grid`，并且每个`grid`中会存储该原子类型在网格中所有体素中与所有可能发生相互作用的刚性原子的能量和
5. 根据`m_no_refine`参数决定是否启用精细化计算，若`m_no_refine==false`，则会再创建`non_cache`对象准备用于精确计算

```cpp
void Vina::compute_vina_maps(double center_x, double center_y, double center_z, double size_x, double size_y, double size_z, double granularity, bool force_even_voxels) {
	// Setup the search box
	// Check first that the receptor was added
	if (m_sf_choice == SF_AD42) {
		throw vina_runtime_error("Cannot compute Vina affinity maps using the AD4 scoring function.");
	} else if (!m_receptor_initialized) {
		// m_model
		throw vina_runtime_error("Cannot compute Vina or Vinardo affinity maps. The (rigid) receptor was not initialized.");
	} else if (size_x <= 0 || size_y <= 0 || size_z <= 0) {
		throw vina_runtime_error("Grid box dimensions must be greater than 0 Angstrom");
	} else if (size_x * size_y * size_z > 27e3) {
		std::cerr << "WARNING: Search space volume is greater than 27000 Angstrom^3 (See FAQ)\n";
	}

	// 初始化网格参数
	grid_dims gd;
	vec span(size_x, size_y, size_z);			// 网格盒子尺寸
	vec center(center_x, center_y, center_z);	// 网格中心
	const fl slope = 1e6; // FIXME: too large? used to be 100

	// atom_types保存所有可移动原子的原子类型（若配体已初始化也会包括配体的原子）
	szv atom_types;
	atom_type::t atom_typing = m_scoring_function->get_atom_typing();
	if (m_ligand_initialized)
		atom_types = m_model.get_movable_atom_types(atom_typing);
	else
		atom_types = m_scoring_function->get_atom_types();

	// 根据参数初始化网格的三维维度
	VINA_FOR_IN(i, gd) {
		gd[i].n_voxels = sz(std::ceil(span[i] / granularity));

		// If odd n_voxels increment by 1
		if (force_even_voxels && (gd[i].n_voxels % 2 == 1))
			// because sample points (npts) == n_voxels + 1
			gd[i].n_voxels += 1;

		fl real_span = granularity * gd[i].n_voxels;
		gd[i].begin = center[i] - real_span / 2;
		gd[i].end = gd[i].begin + real_span;
	}

	// 初始化预计算（原子类型）
	precalculate precalculated_sf(*m_scoring_function);
	// Store it now in Vina object because of non_cache
	m_precalculated_sf = precalculated_sf;

	if (m_sf_choice == SF_VINA)
		doing("Computing Vina grid", m_verbosity, 0);
	else
		doing("Computing Vinardo grid", m_verbosity, 0);

	// 计算网格对象
	cache grid(gd, slope);
	grid.populate(m_model, precalculated_sf, atom_types);

	done(m_verbosity, 0);

	// 创建用于精确计算的non_cache对象
	if (!m_no_refine) {
		non_cache nc(m_model, gd, &m_precalculated_sf, slope);
		m_non_cache = nc;
	}

	// Store in Vina object
	m_grid = grid;
	m_map_initialized = true;
}
```

## PS: 预计算实现

上次已经讲过，在这里不再讲了

## `igrid`接口
```cpp
struct igrid {
	virtual fl eval(const model& m, fl v) const = 0;
	virtual fl eval_intra(model& m, fl v) const = 0;
	virtual fl eval_deriv(model& m, fl v) const = 0;
};
```

- `cache`和`non_cache`都继承此接口
- `eval`和`eval_intra`的区别在于，后者只考虑受体，不会计算任何配体的原子
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
### grid

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

---
## non_cache

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

---
# 全局搜索

`Vina::global_search(const int exhaustiveness, const int n_poses, const double min_rmsd, const int max_evals)`，大致过程如下：
1. 初始化搜索参数
2. 调用`parallel_mc(const model& m, output_container& out, const precalculate_byatom& p, const igrid& ig, const vec& corner1, const vec& corner2, rng& generator, std::function<void(double)>* progress_callback) const`，这是一个`()`运算符的重载。得到输出构象
3. 去除冗余构象
4. 计算各个构象的结合能，并计算分数，按照分数排序输出

## 并行框架

这部分是关于Autodock Vina怎么实现多线程并行工作的，主要是使用了boost的相关多线程库
### parallel_mc

搜索的核心部分就是`parallel_mc`结构体，它保存执行的任务与线程数，以及monte-carlo搜索的设置。`global_steps`和`local_steps`根据自由度启发性设置，其他字段都由程序参数决定：
```cpp
struct parallel_mc {
    monte_carlo mc;
    sz num_tasks;
    sz num_threads;
    bool display_progress;

    parallel_mc() : num_tasks(8), num_threads(1), display_progress(true) {}
    void operator()(const model& m, output_container& out, const precalculate_byatom& p, const igrid& ig, const vec& corner1, const vec& corner2, rng& generator, std::function<void(double)>* progress_callback) const;
};

struct monte_carlo {
    unsigned max_evals;
	unsigned global_steps;
	fl temperature;
	vec hunt_cap;
	fl min_rmsd;
	sz num_saved_mins;
	fl mutation_amplitude;
	unsigned local_steps;
    monte_carlo() : max_evals(0), global_steps(2500), temperature(1.2), hunt_cap(10, 1.5, 10), min_rmsd(0.5), num_saved_mins(50), mutation_amplitude(2) {}
	output_type operator()(model& m, const precalculate_byatom& p, const igrid& ig, const vec& corner1, const vec& corner2, incrementable* increment_me, rng& generator) const;
	void operator()(model& m, output_container& out, const precalculate_byatom& p, const igrid& ig, const vec& corner1, const vec& corner2, incrementable* increment_me, rng& generator) const;
};
```

除此以外，`parallel_mc_task`管理单个搜索任务，包括该任务的对接模型副本和搜索的结果容器；`parallel_mc_aux`保存一些搜索时的具体设置，并通过`parallel_mc_task`执行单个搜索任务
```cpp
struct parallel_mc_task {
	model m;
	output_container out;
	rng generator;
	parallel_mc_task(const model& m_, int seed) : m(m_), generator(static_cast<rng::result_type>(seed)) {}
};

struct parallel_mc_aux {
	const monte_carlo* mc;
	const precalculate_byatom* p;
	const igrid* ig;
	const vec* corner1;
	const vec* corner2;
	parallel_progress* pg;
	parallel_mc_aux(const monte_carlo* mc_, const precalculate_byatom* p_, const igrid* ig_, const vec* corner1_, const vec* corner2_, parallel_progress* pg_)
		: mc(mc_), p(p_), ig(ig_), corner1(corner1_), corner2(corner2_), pg(pg_) {}
	// 执行monte-carlo搜索
	void operator()(parallel_mc_task& t) const {
		(*mc)(t.m, t.out, *p, *ig, *corner1, *corner2, pg, t.generator);
	}
};
```

最终执行时：
```cpp
void parallel_mc::operator()(const model& m, output_container& out, const precalculate_byatom& p, const igrid& ig, const vec& corner1, const vec& corner2, rng& generator, std::function<void(double)>* progress_callback) const {
	// 初始化进度管理器
	parallel_progress pp (progress_callback);
	// 创建辅助执行器，传递所有必要的参数
	parallel_mc_aux parallel_mc_aux_instance(&mc, &p, &ig, &corner1, &corner2, (display_progress ? (&pp) : NULL));
	// 创建任务容器并填充任务
	parallel_mc_task_container task_container;
	VINA_FOR(i, num_tasks)
		task_container.push_back(new parallel_mc_task(m, random_int(0, 1000000, generator)));
	// 如果需要显示进度，初始化进度计数器
	if(display_progress) 
		pp.init(num_tasks * mc.global_steps);
	// 创建并行迭代器并执行所有任务
	parallel_iter<parallel_mc_aux, parallel_mc_task_container, parallel_mc_task, true> parallel_iter_instance(&parallel_mc_aux_instance, num_threads);
	parallel_iter_instance.run(task_container);
	// 合并所有任务的结果到最终输出容器
	merge_output_containers(task_container, out, mc.min_rmsd, mc.num_saved_mins);
}
```

### paralell.h

并行线程的执行模式分为同步和异步模式，区别在于：
- 同步模式所有线程动态竞争下一个任务
- 异步模式则是每个线程按照固定的步长获取下一个任务，不会产生竞争
代码中虽然同时定义了异步模式和同步模式，但是最终使用的只有同步模式

```cpp
//	parallel_iter<parallel_mc_aux, parallel_mc_task_container, parallel_mc_task, true> parallel_iter_instance(&parallel_mc_aux_instance, num_threads);
//	parallel_iter_instance.run(task_container);
template<typename F, typename Container, typename Input, bool Sync = false>
struct parallel_iter { 
	parallel_iter(const F* f, sz num_threads) : a(f), pf(&a, num_threads) {}
	void run(Container& v) {
		a.v = &v;
		pf.run(v.size());
	}
private:
	struct aux {
		const F* f;
		Container* v;
		aux(const F* f) : f(f), v(NULL) {}
        /**
         * @brief 处理指定索引的容器元素
         * @param i 元素索引
         */
		void operator()(sz i) const { 
			VINA_CHECK(v);
			(*f)((*v)[i]); 
		}
	};
    aux a;                           ///< 辅助对象实例
    parallel_for<aux, Sync> pf;      ///< 并行for循环对象
};
```


```cpp
template<typename F>
struct parallel_for<F, true> : private boost::thread_group {
	parallel_for(const F* f, sz num_threads) : m_f(f), destructing(false), size(0), started(0), finished(0) {
		a.par = this; // VC8 warning workaround
        VINA_FOR(i, num_threads)
            create_thread(boost::ref(a));   // 创建工作线程，调用a.par->loop()
    }
	void run(sz size_) {
		boost::mutex::scoped_lock self_lk(self);
        size = size_;
        finished = 0;
        started = 0;
        cond.notify_all(); // many things modified
        while(finished < size) // wait until processing of all elements is finished
            busy.wait(self_lk);
    }
    virtual ~parallel_for() {
        {
			boost::mutex::scoped_lock self_lk(self);
            destructing = true;
            cond.notify_all(); // destructing modified
        }
        join_all(); 
    }
private:
    void loop() {
        while(boost::optional<sz> i = get_next()) {
			(*m_f)(i.get());
			{
				boost::mutex::scoped_lock self_lk(self);
				++finished;         // 增加已完成任务计数
				busy.notify_one();  // 通知主线程
			}
		}
    }
    struct aux {
        parallel_for* par;
        aux() : par(NULL) {}
		void operator()() const { par->loop(); }
    };

    aux a;                          ///< 辅助对象实例
    const F* m_f;                   ///< 函数对象指针（parallel_mc_aux::operator()(parallel_mc_task& t)）
    boost::condition cond;          ///< 工作线程同步条件变量
    boost::condition busy;          ///< 主线程通知条件变量
    bool destructing;               ///< 析构标志
    sz size;                        ///< 任务总数
    sz started;                     ///< 已开始的任务数量
    sz finished;                    ///< 已完成的任务数量
    boost::mutex self;              ///< 互斥锁
    /**
     * @brief 获取下一个任务索引（线程安全）
     * @return 下一个任务索引的可选值，如果正在析构则返回空
     */
	boost::optional<sz> get_next() {
		boost::mutex::scoped_lock self_lk(self);
        while(!destructing && started >= size)
            cond.wait(self_lk);
		if(destructing) return boost::optional<sz>(); // NOTHING
		sz tmp = started;
        ++started;
        return tmp;
    }
};
```


```cpp
    parallel_iter<parallel_mc_aux, parallel_mc_task_container, parallel_mc_task, true> parallel_iter_instance(&parallel_mc_aux_instance, num_threads);
    parallel_iter_instance.run(task_container);
```
对于上面这两行代码，实际流程是：
1. 实例化`parallel_iter`，构造时会同时实例化字段`aux a`和`parallel_for<aux, Sync> pf`
2. `pf`的实例化会创建所有工作线程`create_thread(boost::ref(a))`，每个线程都在循环`loop`中
3. `parallel_iter_instance.run`->`pf.run`->设置任务数量并让工作线程开始工作

---

## 全局搜索

1. 随机初始化构象，设置局部优化器的搜索步数
2. 全局搜索迭代：
	1. 突变构象
	2. 在小范围内进行局部优化（`hunt_cap=(10,10,10)`）
	3. 若迭代次数用完或者满足metropolis准则，则接受新构象
	4. 若新构象的结合能比最佳结合能还小，则在取消搜索范围限制的情况下进行进一步的优化；更新坐标，把新的构象放入输出容器
```cpp
void monte_carlo::operator()(model& m, output_container& out, const precalculate_byatom& p, const igrid& ig, const vec& corner1, const vec& corner2, incrementable* increment_me, rng& generator) const {
    int evalcount = 0;
	vec authentic_v(1000, 1000, 1000);
	conf_size s = m.get_size();
	change g(s);
	// 随机初始化构象
	output_type tmp(s, 0);
	tmp.c.randomize(corner1, corner2, generator);
	fl best_e = max_fl;	 // 找到的最佳能量
	// 局部优化器
	quasi_newton quasi_newton_par;
    quasi_newton_par.max_steps = local_steps;
	// 全局搜索迭代
	VINA_U_FOR(step, global_steps) {
		if(increment_me)
			++(*increment_me);
		if((max_evals > 0) & (evalcount > max_evals))
			break;
		// 对构象进行突变
		output_type candidate = tmp;
		mutate_conf(candidate.c, m, mutation_amplitude, generator);
		// 局部优化, hunt_cap=(10,10,10)
		quasi_newton_par(m, p, ig, candidate, g, hunt_cap, evalcount);

		if(step == 0 || metropolis_accept(tmp.e, candidate.e, temperature, generator)) {
			tmp = candidate;	// 接受新构象

			// 结合能小于最佳能量，精细筛选
			if(tmp.e < best_e || out.size() < num_saved_mins) {
				// 局部优化, authentic_v=(1000,1000,1000)，相当于取消了搜索的范围限制，允许大幅度构象变化
				quasi_newton_par(m, p, ig, tmp, g, authentic_v, evalcount);
				tmp.coords = m.get_heavy_atom_movable_coords();
				add_to_output_container(out, tmp, min_rmsd, num_saved_mins); // 20 - max size
				if(tmp.e < best_e)
					best_e = tmp.e;
			}
		}
	}
	VINA_CHECK(!out.empty());
	VINA_CHECK(out.front().e <= out.back().e); // make sure the sorting worked in the correct order
}
```
### PS：构象相关

1. 提供扭转角的操作函数：置0、随机生成、按比例增量更新、检查是否过于接近等
2. 定义构象与构象的变化，包括刚体、配体和柔性残基
#### 刚体

在刚体变化量中，刚体朝向的变化量选择使用了三元组；而在刚体构象中，选择使用了四元数。

```cpp
struct rigid_change {
    vec position;
    vec orientation;
    rigid_change() : position(0, 0, 0), orientation(0, 0, 0) {}
};
struct rigid_conf {
    vec position;
    qt orientation;
    rigid_conf() : position(0, 0, 0), orientation(qt_identity) {}
	void increment(const rigid_change& c, fl factor) {
		position += factor * c.position;
		vec rotation; rotation = factor * c.orientation;
		quaternion_increment(orientation, rotation); // 将角度转换为四元数
	}
};
```

#### 配体
在刚体的6个自由度的基础上添加扭转角
```cpp
struct ligand_change {
	rigid_change rigid;
	flv torsions;
};
struct ligand_conf {
	rigid_conf rigid;
	flv torsions;
};
```

#### 柔性残基
能活动的部分只有扭转角
```cpp
struct residue_change {
	flv torsions;
};
struct residue_conf {
	flv torsions;
};
```

#### 总体

- 由配体和柔性残基的构象/变化向量组成
- 由`conf_size`初始化

```cpp
/// @brief 获取模型的配体和柔性残基的扭转角总数 
conf_size model::get_size() const {
    conf_size tmp;
    tmp.ligands = ligands.count_torsions();  // 统计所有配体的扭转角总数
    tmp.flex    = flex   .count_torsions();  // 统计所有柔性残基的扭转角总数
    return tmp;
}
struct conf_size {
	szv ligands;	///< 每个配体的扭转角数量向量
	szv flex;		///< 每个柔性残基（受体）的扭转角数量向量
	/// @brief 计算总的自由度
	sz num_degrees_of_freedom() const {
		return sum(ligands) + sum(flex) + 6 * ligands.size();
	}
};
struct change {
    std::vector<ligand_change> ligands;
    std::vector<residue_change> flex;
    /// @brief 构造函数，根据conf_size初始化
	change(const conf_size& s) : ligands(s.ligands.size()), flex(s.flex.size()) {
		VINA_FOR_IN(i, ligands)
			ligands[i].torsions.resize(s.ligands[i], 0);
		VINA_FOR_IN(i, flex)
			flex[i].torsions.resize(s.flex[i], 0);
	}
};
struct conf {
	std::vector<ligand_conf> ligands;	///< 配体构象向量
	std::vector<residue_conf> flex;		///< 柔性残基构象向量
	/// @brief 构造函数，根据尺寸规格初始化
	conf(const conf_size& s) : ligands(s.ligands.size()), flex(s.flex.size()) {
		VINA_FOR_IN(i, ligands)
			ligands[i].torsions.resize(s.ligands[i], 0); // FIXME?
		VINA_FOR_IN(i, flex)
			flex[i].torsions.resize(s.flex[i], 0); // FIXME?
	}
};
```

#### 输出结果

```cpp
struct output_type {
	conf c;                ///< 构象
	fl e;                  ///< 结合能
	fl lb;                 ///< 下界能量
	fl ub;                 ///< 上界能量
	fl intra;              ///< 分子内能量
	fl inter;              ///< 分子间能量
	fl conf_independent;   ///< 构象无关能量
	fl unbound;            ///< 未结合状态能量
	fl total;              ///< 总能量
	vecv coords;           ///< 原子坐标

	output_type(const conf& c_, fl e_) : c(c_), e(e_) {}
};
```
## 局部优化器

### quasi_newton

```cpp
struct quasi_newton {
    unsigned max_steps;
    fl average_required_improvement;
    quasi_newton() : max_steps(1000), average_required_improvement(0.0) {}
    // clean up
    void operator()(model& m, const precalculate_byatom& p, const igrid& ig, output_type& out, change& g, const vec& v, int& evalcount) const; // g must have correct size
};

struct quasi_newton_aux {
    model* m;
    const precalculate_byatom* p;
    const igrid* ig;
    const vec v;

    quasi_newton_aux(model* m_, const precalculate_byatom* p_, const igrid* ig_, const vec& v_) : m(m_), p(p_), ig(ig_), v(v_) {}
    
    fl operator()(const conf& c, change& g) {
        // Before evaluating conf, we have to update model
        m->set(c);
        const fl tmp = m->eval_deriv(*p, *ig, v, g);
        return tmp;
    }
};

void quasi_newton::operator()(model& m, const precalculate_byatom& p, const igrid& ig, output_type& out, change& g, const vec& v, int& evalcount) const { // g must have correct size
    quasi_newton_aux aux(&m, &p, &ig, v);

    fl res = bfgs(aux, out.c, g, max_steps, average_required_improvement, 10, evalcount);

    // Update model a last time after optimization
    m.set(out.c);
    out.e = res;
}
```


### bfgs

1. 初始化Hessian矩阵近似为单位矩阵；复制初始构象和梯度；存档原始状态
2. 计算搜索方向 p = -H * g
3. 线搜索确定步长
4. 更新位置和梯度
5. 更新Hessian逆矩阵近似
6. 检查收敛条件

