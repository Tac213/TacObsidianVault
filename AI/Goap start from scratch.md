Goap即Goal Oriented Action Planning，是一种基于目标的行为规划器，其核心算法是从目标开始逆向搜索行为链。

Github有个[基于Unity的Goap开源项目](https://github.com/sploreg/goap)，这个项目里实现了一版[搜索算法](https://github.com/sploreg/goap/blob/master/Assets/Standard%20Assets/Scripts/AI/GOAP/GoapPlanner.cs)，核心函数是buildGraph。不过他实现的是正向搜索，从start找到goal，和逆向搜索的预期不符，而且因为正向搜索无法像逆向搜索那样从对action的effect能否达成当前待满足的世界状态做筛选，会导致搜索过程中产生的分支比逆向搜索多，从而效率不如逆向搜索。

尝试自己写了一版基于a star的逆向搜索算法，不过由于没有实际需求都是自己YY的测试用例，所以可能存在考虑不合理的地方，暂且记下笔记，日后实际用上时再翻看。

## Action定义

Action定义很简单，每个action有个cost，用来作为a start节点的g值，而每个action又有一个前提world state的效果world state。

```python
class GOAPAction(object):

    def __init__(self):
        self.cost = 0
        self.name = self.__class__.__name__
        self.preconditions = None
        self.effects = None

    def __eq__(self, action):
        """
        判断两个Action是否相等
        Args:
            action: GOAPAction
        Returns:
            bool
        """
        return self.name == action.name

    def check_proceduarl_precondition(self, entity):
        """
        程序上的预设条件
        Args:
            entity
        Returns:
            bool
        """
        return True

    def __bool__(self):
        return self.__class__ is not GOAPAction

```

## WorldState定义

在这版实现里，WorldState是个非常重要的对象，关乎如何寻找相邻节点、如何退出搜索循环等等。

WorldState被认为是一个atom的集合。既然是集合，就存在各种运算方法：是否包含、求并集、求交集、求差集。

在实现过程中，遇到的最大的疑问就是：当两个WorldState有同一个key，但是value不同时，如何求并集？比如下面2个WorldState:

- WorldState1: `{'a': True, 'b': False}`
- WorldState2: `{'a': False}`

目前是谁是self，就以谁的value为准，比如上面2个WorldState求并集的结果是`{'a': True, 'b': False}`。

实现如下：

```python
class GOAPWorldState(object):

    def __init__(self, atoms=None):
        self.atoms = atoms or {}

    def copy(self):
        """
        返回世界状态的拷贝
        Returns:
            GOAPWorldState
        """
        ws = GOAPWorldState()
        for name, value in self.iter_atoms():
            ws.add_atom(GOAPAtom(name, value))
        return ws

    def add_atom(self, atom):
        """
        添加单个世界状态元素
        Args:
            atom: Atom
        Returns:
            None
        """
        self.atoms[atom.name] = atom.value

    def is_included(self, w):
        """
        指定世界状态是否包含当前世界状态
        Args:
            w: GOAPWorldState
        Returns:
            bool
        """
        for name, value in w.iter_atoms():
            v = self.atoms.get(name, None)
            if v is None or v != value:
                return False
        return True

    def iter_atoms(self):
        """
        遍历所有Atom
        Returns:
            generator
        """
        for name, value in self.atoms.items():
            yield name, value

    def join_world_state(self, effects):
        """
        对世界状态造成影响
        Args:
            effects: GOAPWorldState
        Returns:
            None
        """
        for name, value in effects.iter_atoms():
            self.atoms[name] = value

    def difference(self, another_world_state):
        """
        返回self与另一个world state的差集
        Args:
            another_world_state: WorldState
        Returns:
            WorldState
        """
        result = self.__class__()
        for name, value in self.iter_atoms():
            v = another_world_state.atoms.get(name)
            if v is None or v != value:
                result.atoms[name] = value
        return result

    def symmetric_difference(self, another_world_state):
        """
        返回self和另一个world state不重复的部分
        相当于(self - another) | (another - self)
        如果self和another存在key相同但是value不同的atom, 则以self为准
        Args:
            another_world_state: WorldState
        Returns:
            WorldState
        """
        result = self.__class__()
        for name, value in another_world_state.iter_atoms():
            v = self.atoms.get(name)
            if v is None or v != value:
                result.atoms[name] = value
        for name, value in self.iter_atoms():
            v = another_world_state.atoms.get(name)
            if v is None or v != value:
                result.atoms[name] = value
        return result

    def intersection(self, another_world_state):
        """
        返回self与另一个world state的交集
        Args:
            another_world_state: WorldState
        Returns:
            WorldState
        """
        result = self.__class__()
        for name, value in another_world_state.iter_atoms():
            v = self.atoms.get(name)
            if v == value:
                result.atoms[name] = value
        return result

    def union(self, another_world_state):
        """
        返回self与另一个world state的并集
        如果存在相同key但不同value的atom, 以self为准
        Args:
            another_world_state: WorldState
        Returns:
            WorldState
        """
        result = self.copy()
        for name, value in another_world_state.iter_atoms():
            v = self.atoms.get(name)
            if v is not None and v != value:
                result.atoms[name] = v
                continue
            result.atoms[name] = value
        return result

    def __len__(self):
        return len(self.atoms)

    def has_same_key_but_different_atom_with(self, another_world_state):
        for name, value in another_world_state.iter_atoms():
            v = self.atoms.get(name)
            if v is not None and v != value:
                return True
        return False

```

## A star node

由于当前算法基于a star，所以还需要实现一个a star的节点，每个节点会对应一个action。

根据a star算法，每个节点都有f和g和h，其中`f = g + h`，`g`是action的cost，`h`目前比较随意，是当前节点待满足的世界状态和goal的差别。

每个节点会有一个待满足的世界状态，当**待满足的世界状态是当前世界状态的子集**时，说明找到了起始节点。

每个节点还会有一个后续所有效果的世界状态总和，用于处理某些动作存在额外效果的情况，这个会在后文描述。

具体实现：

```python
class GOAPNode(object):

    def __init__(self, world_state: GOAPWorldState, action, all_effects: GOAPWorldState = None):
        self.h = 0
        self.g = action.cost
        self.parent = None
        self.pending_world_state = world_state
        self.action = action
        self.all_effects = all_effects or GOAPWorldState()

    def set_h(self, w):
        self.h = len(self.pending_world_state.difference(w))

    def set_g(self, node):
        self.g += node.g

    @property
    def f(self) -> int:
        return self.g + self.h
```

## 算法实例

在实现算法之前，最好先看几个实例，帮助编码。示例：

- 目标：拿到金币`{'GetCoin': True}`
- 当前世界状态：没有钥匙、没有开门、没有拿到金币`{'GetCoin': False, 'GetKey': False, 'OpenDoor': False}`
- 世界规则：金币在门后面，开门需要钥匙，钥匙只有1把，金币也只有1个
- 门可以通过钥匙打开，也可以直接把门破坏，但是把门破坏的消耗会很大
- 地图上有钥匙的话才能拿到钥匙

根据这些世界规则，action定义如下：

```python
class BreakDoor(GOAPAction):

    def __init__(self):
        super().__init__()
        self.cost = 10  # higher than OpenDoor
        self.preconditions = GOAPWorldState({'OpenDoor': False})
        self.effects = GOAPWorldState({'OpenDoor': True})


class OpenDoor(GOAPAction):

    def __init__(self):
        super().__init__()
        self.cost = 1
        self.preconditions = GOAPWorldState({'HaveKey': True, 'OpenDoor': False})
        self.effects = GOAPWorldState({'OpenDoor': True, 'HaveKey': False})


class GetKey(GOAPAction):

    def __init__(self):
        super().__init__()
        self.cost = 1
        self.preconditions = GOAPWorldState({'HaveKey': False})
        self.effects = GOAPWorldState({'HaveKey': True})

    def check_proceduarl_precondition(self, entity):
        world = entity.world
        return len(world.entities_by_type.get('Key', {})) > 0


class GetCoin(GOAPAction):

    def __init__(self):
        super().__init__()
        self.cost = 1
        self.preconditions = GOAPWorldState({'GetCoin': False, 'OpenDoor': True})
        self.effects = GOAPWorldState({'GetCoin': True})
```

算法过程：

1. 构建一个初始的goal节点，该节点待满足的世界状态为`{'GetCoin': True}`
2. 遍历可以使用的action，找到effects和待满足的世界状态**存在交集**的action，结果为`[GetCoin]`，遍历之为每个action构建节点，作为当前节点的相邻节点
3. 以当前节点待满足的世界状态作为基础，确定每个新节点的待满足的世界状态，首先将新节点对应action的effects从当前节点待满足的世界状态里划掉(作**差集**)，结果为`{}`，然后将新节点对应的action的前提条件和刚才差集的结果做**并集**，并集的结果就是新节点待满足的世界状态，结果为`{'GetCoin': False, 'OpenDoor': True}`
4. 对新节点重复2 - 3这个过程，存在交集的action为`[BreakDoor, OpenDoor]`，会创建2个新节点，每个新节点待满足的世界状态分别为`{'GetCoin': False, 'OpenDoor': False}`和`{'GetCoin': False, 'OpenDoor': False, 'HaveKey': True}`
5. 在2个新节点里找到f值最小的节点，结果为后者，对这个节点重复2 - 3这个过程，存在交集的action为`[GetKey]`，会创建1个新节点，该新节点待满足的世界状态为`{'GetCoin': False, 'OpenDoor': False, 'HaveKey': False}`，该世界状态已经是当前世界状态的子集，找到了通路`[GetKey, OpenDoor, GetCoin]`
6. 如果世界中没有钥匙，5会跑不通，此时会找`BreakDoor`对应的节点，对这个节点重复2 - 3这个过程，尽管已经没有存在交集的action了，但是该节点待满足的世界状态`{'GetCoin': False, 'OpenDoor': False}`已经是当前世界状态的子集，所以也找到了通路`[BreakDoor, GetCoin]`

整个算法过程最为复杂的是第3步，比如当前待满足的世界状态是`{'GetCoin': False, 'OpenDoor': True}`，待处理的action是`OpenDoor`，待处理action对应的待满足的世界状态计算过程如下图：

![pending_world_state计算过程](../Images/goap_calc_pending_world_state.png)

上面这个示例中有一种情况没有考虑到，比如：

goal: {'a': True, 'b': False}

action1的effects: {'a': True, 'b': True}

action2的effects: {'b': False}

此时计算结果应为`[action1, action2]`。

但我们在考虑将action1加到相邻节点时，会发现直接从action1也能达到goal的一部分，但action1会有**额外的效果**{'b': True}，和goal存在**相同key但value不同**的atom，所以不能直接从action1到goal。

那么如何将这种情况也考虑上呢？可以看下下面这个示例：

- 目标：建造房子，而且手上有木材，`{'BuildHouse': True, 'GetWood': True}`
- 当前世界状态：有木材，没有房子，`{'BuildHouse': False, 'GetWood': True}`
- 世界规则：建造房子需要消耗木材，可以捡木材

根据这个世界规则，action定义如下：

```python
class BuildHouse(GOAPAction):

    def __init__(self):
        super().__init__()
        self.cost = 10
        self.preconditions = GOAPWorldState({'BuildHouse': False, 'GetWood': True})
        self.effects = GOAPWorldState({'BuildHouse': True, 'GetWood': False})


class GetWood(GOAPAction):

    def __init__(self):
        super().__init__()
        self.cost = 1
        self.preconditions = GOAPWorldState()
        self.effects = GOAPWorldState({'GetWood': True})
```

算法过程：

1. 构建一个初始的goal节点，该节点待满足的世界状态为`{'BuildHouse': True, 'GetWood': True}`
2. 遍历可以使用的action，找到effects和待满足的世界状态**存在交集**的action，结果为`[BuildHouse, GetWood]`，遍历之为每个action构建节点，作为当前节点的相邻节点
3. 以当前节点待满足的世界状态为基础，确定每个新节点的待满足的世界状态，按照图中的过程进行计算，发现求pending和BuildHouse.effects差集的过程中，会存在`{'GetWood': False}`这个额外的效果，这个效果**在当前已经确定好的节点效果里无法被抵消**，所以不能把BuildHouse作为当前节点的相邻节点，而GetWood动作则不会有这个问题，此时GetWood节点对应的待满足的世界状态为`{'BuildHouse': True}`
4. 对GetWood节点重复2过程，发现可能相邻的节点为`[BuildHouse]`，而在重复3过程时，仍然发现会存在`{'GetWood': False}`这个额外的效果，不过与上一次不同，这个**额外的效果可以被当前的GetWood节点抵消**，所以可以将BuildHouse作为当前节点的相邻节点，此时BuildHouse待满足的世界状态计算结果为`{'BuildHouse': False, 'GetWood': True}`，已经是当前世界状态的子集，所以找到了通路`[BuildHouse, GetWood]`

这次的改进在于，除了计算pending，还应该计算当前节点的**后续所有节点的效果总和**，如果candidate action的额外效果能够被这个总和**抵消**，那么candidate action可以作为相邻节点，否则不行。

这也是为什么每个a star node需要有一个`all_effects`属性的原因。

## 算法代码

根据上面讲述的算法过程，结合a star算法，可以得到下面的代码：

```python
    def generate_plan(self, entity, actions, goal):
        last = None
        is_start_reached = False
        goal_node = GOAPNode(goal, GOAPAction())

        open_list = []  # type: typing.List[GOAPNode]
        closed_list = []  # type: typing.List[GOAPNode]
        open_list.append(goal_node)

        while open_list:
            current_node_index = self.lowest_f_node_index(open_list)
            current_node = open_list.pop(current_node_index)
            closed_list.append(current_node)

            # reach current world state
            if self.current_world_state.is_included(current_node.pending_world_state):
                last = current_node
                is_start_reached = True
                break

            adjacent_nodes = []
            for action in actions:
                if not action.check_proceduarl_precondition(entity):
                    continue
                if current_node.action == action:
                    continue
                if not action.effects.intersection(current_node.pending_world_state):
                    continue
                # pending_world_state = action.effects.symmetric_difference(current_node.pending_world_state)
                other_effects = action.effects.difference(current_node.pending_world_state)
                if other_effects:
                    other_effects.join_world_state(current_node.all_effects)
                    if other_effects.has_same_key_but_different_atom_with(goal):
                        continue
                pending_world_state = current_node.pending_world_state.difference(action.effects)
                pending_world_state = action.preconditions.union(pending_world_state)
                all_effects = current_node.all_effects.copy()
                all_effects.join_world_state(action.effects)
                new_node = GOAPNode(pending_world_state, action, all_effects)
                adjacent_nodes.append(new_node)

            for adjacent_node in adjacent_nodes:
                if adjacent_node in closed_list:
                    # node has already been accessed
                    continue
                if adjacent_node not in open_list:
                    # new node, add to open list and set parent to current node
                    open_list.append(adjacent_node)
                    adjacent_node.parent = current_node
                    adjacent_node.set_g(current_node)
                    adjacent_node.set_h(goal)
                else:
                    temp_g_score = current_node.g + adjacent_node.action.cost
                    if temp_g_score < adjacent_node.g:
                        # it takes less cost, reset parent to current node
                        adjacent_node.parent = current_node
                        adjacent_node.g = temp_g_score

        solution = []

        if is_start_reached:
            plan_node = last
            while plan_node:
                if plan_node.action:
                    solution.append(plan_node.action)
                plan_node = plan_node.parent

        return solution
```
