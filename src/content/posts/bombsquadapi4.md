---
title: BombSquad api4 文档
published: 2025-10-31
pinned: true
description: 简单的一个api4的文档
tags: [Markdown, Blogging]
draft: false
sourceLink: "https://blog.cnmsb.cfd/posts/bombsquadapi4/"
author: sddlol
---

# BombSquad APIv4 - 重建文档
本文档基于 APIv4 MOD 示例代码重建，旨在为 BombSquad MOD 开发者提供参考。
1. 脚本入口点 (Entry Points)
BombSquad 加载 MOD 时会寻找两个关键的 "入口" 函数。
bsGetAPIVersion()
声明你的脚本所使用的 API 版本。对于 APIv4，你必须返回 4。
def bsGetAPIVersion():
    # see bombsquadgame.com/apichanges
    return 4

bsGetGames()
注册你的游戏模式。这个函数必须返回一个包含你的游戏活动（GameActivity）类的列表。
def bsGetGames():
    return [MyCustomGame, AnotherCustomGame]

2. 游戏活动 (Game Activity)
"游戏活动" (Game Activity) 是你游戏模式的核心。它控制着游戏的规则、设置、生命周期和事件。你的游戏类通常应继承自 bs.TeamGameActivity 或 bs.FreeForAllActivity。
import bs

class LastStandwithFriendsGame(bs.TeamGameActivity):
    
    # ... 你的游戏逻辑 ...

2.1. 静态类方法 (Class Methods)
这些方法定义了游戏模式在游戏列表中的显示方式和设置。
 * @classmethod def getName(cls):
   返回游戏模式的显示名称。
   @classmethod
def getName(cls):
    return 'Last Stand with Friends!'

 * @classmethod def getDescription(cls, sessionType):
   返回游戏模式的描述。
   @classmethod
def getDescription(cls, sessionType):
    return 'Play The Last Stand with Friends now! by AbhinaY'

 * @classmethod def getScoreInfo(cls):
   定义游戏结束时如何计分。
   @classmethod
def getScoreInfo(cls):
    return {'scoreName':'Survived',
            'scoreType':'seconds',
            'noneIsWinner':True}

 * @classmethod def supportsSessionType(cls, sessionType):
   声明此游戏模式支持哪种会话类型（如团队、混战）。
   @classmethod
def supportsSessionType(cls,sessionType):
    return True if (issubclass(sessionType,bs.TeamsSession)
                    or issubclass(sessionType,bs.FreeForAllSession)) else False

 * @classmethod def getSupportedMaps(cls, sessionType):
   返回此游戏模式支持的地图名称列表。
   @classmethod
def getSupportedMaps(cls,sessionType):
    return ['Rampage','Courtyard','Doom Shroom']

 * @classmethod def getSettings(cls, sessionType):
   定义游戏模式的设置选项，这些选项会显示在设置菜单中。
   @classmethod
def getSettings(cls,sessionType):
    settings = [
        ("Lives Per Player", {'default':1, 'minValue':1, 'maxValue':10, 'increment':1}),
        ("Epic Mode", {'default':True})
    ]
    return settings

2.2. 游戏生命周期 (Game Lifecycle Methods)
这些实例方法在游戏的不同阶段被调用。
 * __init__(self, settings):
   游戏实例初始化。settings 参数是一个字典，包含了玩家在设置菜单中选择的值。
   def __init__(self, settings):
    bs.TeamGameActivity.__init__(self, settings)
    if self.settings['Epic Mode']: 
        self._isSlowMotion = True
    self.announcePlayerDeaths = True
    self._scoreBoard = bs.ScoreBoard()

 * onTransitionIn(self):
   游戏过渡（淡入）时调用。适合播放音乐。
   def onTransitionIn(self):
    bs.TeamGameActivity.onTransitionIn(self, music='Epic')
    self._startGameTime = bs.getGameTime()

 * onBegin(self):
   游戏正式开始时调用。适合设置计时器、刷怪点和初始化玩家。
   def onBegin(self):
    bs.TeamGameActivity.onBegin(self)
    self.setupStandardTimeLimit(self.settings['Time Limit'])
    self.setupStandardPowerupDrops()

    # 创建一个Bot集合
    self._bots = bs.BotSet()
    # 创建一个TNT生成器
    self._tntSpawner = bs.TNTSpawner(position=(0,5.5,-6))

    # 启动一个重复计时器
    bs.gameTimer(1000, self._update, repeat=True)

 * onTeamJoin(self, team):
   当一个队伍加入游戏时调用。
   def onTeamJoin(self, team):
    # 使用 gameData 字典存储队伍的自定义数据
    team.gameData['survivalSeconds'] = None
    team.gameData['score'] = 0

 * onPlayerJoin(self, player):
   当一个玩家加入游戏时调用。
   def onPlayerJoin(self, player):
    # 使用 gameData 字典存储玩家的自定义数据
    player.gameData['lives'] = self.settings['Lives Per Player']
    # 为玩家创建图标
    player.gameData['icons'] = [Icon(player, position=(0,50), scale=0.8)]
    # 刷出玩家
    if player.gameData['lives'] > 0:
        self.spawnPlayer(player)

 * onPlayerLeave(self, player):
   当一个玩家离开游戏时调用。
   def onPlayerLeave(self, player):
    bs.TeamGameActivity.onPlayerLeave(self, player)
    player.gameData['icons'] = None # 清理数据

 * endGame(self):
   结束游戏。
   def endGame(self):
    if self.hasEnded(): return
    results = bs.TeamGameResults() # 创建结果对象
    for team in self.teams:
        results.setTeamScore(team, team.gameData['score']) # 设置分数
    self.end(results=results) # 调用 self.end() 真正结束游戏
    2.3. 事件处理 (handleMessage)
这是游戏活动中 最重要 的方法之一。它接收并处理游戏中发生的所有事件。
def handleMessage(self, m):
    # --- 玩家死亡事件 ---
    if isinstance(m, bs.PlayerSpazDeathMessage):
        bs.TeamGameActivity.handleMessage(self, m) # 调用父类的方法
        player = m.spaz.getPlayer()
        killer = m.killerPlayer
        
        player.gameData['lives'] -= 1
        
        if killer and killer.exists():
            killer.getTeam().gameData['score'] += 1 # 给击杀者队伍加分
        
        # ... 检查游戏是否结束 ...

    # --- Bot 死亡事件 ---
    elif isinstance(m, bs.SpazBotDeathMessage):
        pts, importance = m.badGuy.getDeathPoints(m.how)
        if m.killerPlayer is not None and m.killerPlayer.exists():
            self.scoreSet.playerScored(m.killerPlayer, pts, kill=True)
            
    # --- 玩家得分事件 ---
    elif isinstance(m, bs.PlayerScoredMessage):
        self._score.add(m.score)
        self._updateScores()

    # --- 其他消息 ---
    else:
        bs.TeamGameActivity.handleMessage(self, m)

常见的消息 (Message) 类型:
 * bs.PlayerSpazDeathMessage: 玩家角色死亡。
   * m.spaz: 死亡的 bs.PlayerSpaz 对象。
   * m.killerPlayer: bs.Player 对象（如果有的话）。
   * m.how: 死亡方式 (e.g., 'impact', 'fall', 'pickup')。
 * bs.SpazBotDeathMessage: Bot 死亡。
 * bs.HitMessage: 任意 Spaz 受到攻击。
 * bs.PlayerScoredMessage: 玩家得分（通常用于夺旗或足球模式）。
 * bs.PickedUpMessage: 物品被捡起。
 * bs.DroppedMessage: 物品被丢下。
3. 玩家与角色 (Player & Spaz)
 * bs.Player: 代表连接到游戏的 "玩家"（即用户）。
 * bs.PlayerSpaz: 代表玩家在游戏中的 "角色"（即那个可以跑动和拳击的小人）。
3.1. 刷出玩家 (Spawning)
spawnPlayer 是 GameActivity 的一个方法。
def spawnPlayer(self, player):
    # 刷出一个标准的玩家角色
    spaz = self.spawnPlayerSpaz(player)
    
    # ... 你可以在这里添加自定义逻辑 ...
    # 例如，连接到自定义的 Spaz 类
    
    # 告诉玩家的图标它已经刷出了
    for icon in player.gameData['icons']:
        icon.handlePlayerSpawned()

3.2. 自定义 Spaz (bs.PlayerSpaz)
你可以通过继承 bs.PlayerSpaz 来创建拥有特殊能力的角色，如 smash.py 中所示。
class PlayerSpaz_Smash(bs.PlayerSpaz):
    multiplyer = 1 # 伤害倍率

    def handleMessage(self, m):
        if isinstance(m, bs.HitMessage):
            # ... 自定义受击逻辑 ...
            
            # 原始伤害
            damage = m.flatDamage if m.flatDamage else (m.magnitude * 0.22)
            
            # 施加伤害倍率
            velocityMag = m.velocityMagnitude * (self.multiplyer ** 1.9)
            
            # 向节点发送 "impulse" 消息来施加力
            self.node.handleMessage("impulse", m.pos[0], m.pos[1], m.pos[2],
                                    m.velocity[0], m.velocity[1], m.velocity[2],
                                    m.magnitude, velocityMag, m.radius, 0,
                                    m.forceDirection[0], m.forceDirection[1], m.forceDirection[2])
            
            # 增加倍率
            self.multiplyer += min(damage / 2000, 0.15)
            
        elif isinstance(m, bs.DieMessage):
            # ... 自定义死亡逻辑 ...
            self.oob_effect() # 播放一个自定义的屏幕外特效
            super(self.__class__, self).handleMessage(m)
            
        elif isinstance(m, bs.PowerupMessage):
            if m.powerupType == 'health':
                # ... 自定义吃血包逻辑 (在 Smash 模式中是减伤) ...
                self.multiplyer *= 0.75
            else:
                super(self.__class__, self).handleMessage(m)
        else:
            super(self.__class__, self).handleMessage(m)

在你的 GameActivity 中，你需要重写 spawnPlayer 来使用这个自定义的 Spaz：
# 在你的 GameActivity 类中:
def spawnPlayer(self, player):
    # ... 获取位置 ...
    
    # 实例化你的自定义 Spaz
    spaz = PlayerSpaz_Smash(color=player.color,
                            highlight=player.highlight,
                            character=player.character,
                            player=player)
    
    # 关键：将 Player 和 Actor (Spaz) 关联起来
    player.setActor(spaz) 
    
    # 连接玩家的输入控制
    spaz.connectControlsToPlayer()
    
    # 发送消息让 Spaz 站立在刷出点
    spaz.handleMessage(bs.StandMessage(position))

4. Actors, Nodes 和 UI
BombSquad 的世界由 "Actors" 和 "Nodes" 构成。
 * Node (节点): 游戏引擎中的底层对象，如一个图像、一个文本、一个物理实体或一个光源。
 * Actor (演员): 一个 Python 对象 (一个类实例)，它封装了一个或多个 Node，并为其添加逻辑。bs.PlayerSpaz 和 bs.Bomb 都是 Actor。
4.1. bs.Actor
所有 "活" 的对象都应该继承自 bs.Actor。
class Icon(bs.Actor):

    def __init__(self, player, position, scale):
        bs.Actor.__init__(self) # 必须调用父类的 init
        
        self._player = player
        
        # 核心：创建一个 Node
        # self.node 是这个 Actor 的主 Node
        self.node = bs.newNode('image',
                               owner=self,
                               attrs={'texture': player.getIcon()['texture'],
                                      'tintColor': player.getIcon()['tintColor'],
                                      'opacity': 1.0,
                                      'absoluteScale': True,
                                      'attach': 'bottomCenter'})
        
# 你可以创建更多的 Node
        self._nameText = bs.newNode('text',
                                    owner=self.node, # 附加到主 Node 上
                                    attrs={'text': player.getName(),
                                           'color': bs.getSafeColor(player.getTeam().color),
                                           'hAlign': 'center',
                                           'vAlign': 'center'})

    def handlePlayerSpawned(self):
        if not self.node.exists(): return
        self.node.opacity = 1.0

    def handlePlayerDied(self):
        if not self.node.exists(): return
        # 节点动画
        bs.animate(self.node, 'opacity', {0: 1.0, 500: 0.2})
        
    def handleMessage(self, m):
        if isinstance(m, bs.DieMessage):
            if self.node.exists():
                self.node.delete()
        else:
            super(Icon, self).handleMessage(m)
            4.2. bs.newNode(type, attrs={...})
创建所有游戏对象的工厂函数。
 * 常用类型:
   * 'image': 2D 图像 (用于 UI)。
   * 'text': 2D/3D 文本。
   * 'light': 光源。
   * 'spaz': 物理角色实体 (通常由 bs.PlayerSpaz 内部管理)。
   * 'prop': 物理道具。
   * 'math': 一个 "数学" 节点，用于连接和转换其他节点的属性。
4.3. 属性连接 (Attribute Connection)
这是一个高级但强大的功能。你可以将一个节点的输出属性连接到另一个节点的输入属性。smash.py 中的 PowBox 展示了这一点：
# 将 "math" 节点的输出连接到 "text" 节点的 "position"
# 这样，文本就会始终跟随在炸弹上方 0.7 米处。

# 1. 创建一个 "math" 节点用于计算位置
m = bs.newNode('math', owner=self.node, attrs={'input1': (0, 0.7, 0), 'operation': 'add'})

# 2. 将 "bomb" 节点的 "position" 作为 "math" 节点的输入
self.node.connectAttr('position', m, 'input2')

# 3. 创建 "text" 节点
self._powText = bs.newNode('text',
                           owner=self.node,
                           attrs={'text': 'POW!', 'inWorld': True, ...})

# 4. 将 "math" 节点的 "output" 连接到 "text" 节点的 "position"
m.connectAttr('output', self._powText, 'position')

4.4. bs.NodeActor
这是一个 bs.Actor 的便捷包装器，它只包含一个单独的 Node。Duel.py 中用它来显示 "VS" 文本。
self._vsText = bs.NodeActor(bs.newNode("text",
                                       attrs={
                                          'position':(0,105),
                                          'hAlign':"center",
                                          'text':bs.Lstr(resource='vsText')
                                      }))

5. 核心工具 (Core Utilities)
5.1. 计时器 (Timers)
 * bs.gameTimer(duration, callback):
   在 duration 毫秒后调用 callback。
   # 600ms 后调用 self.updateForLives
bs.gameTimer(600, self.updateForLives)

 * bs.Timer(duration, callback, repeat=False):
   功能更全的计时器，可以设置 repeat=True。
   # 每 1000ms (1秒) 调用一次 self._update
self._botUpdateTimer = bs.Timer(1000, bs.WeakCall(self._update), repeat=True)

 * bs.WeakCall(callback):
   (推荐) 当与类方法一起使用时，bs.WeakCall 可以防止循环引用导致的内存泄漏。
   5.2. 动画 (bs.animate)
用于平滑地改变一个 Node 的属性。
# 在 {时间: 值} 字典中定义关键帧
# 0ms: 1.0, 50ms: 0.0, 100ms: 1.0, 150ms: 0.0, ...
bs.animate(self.node, 'opacity', {0:1.0, 50:0.0, 100:1.0, 150:0.0, ...})

# 循环动画 (来自 Duel.py)
bs.animate(c, 'input0', {0:tint[0], 10000:tint[0]*1.2, 20000:tint[0]}, loop=True)

5.3. 音效和特效
 * bs.getSound(name): 加载一个声音。
 * bs.playSound(sound, volume=1.0, position=None): 播放声音。
   bs.playSound(bs.getSound('cheer'))

 * bs.emitBGDynamics(...): 释放粒子效果 (火花、汗水、碎片等)。
   # (来自 smash.py 的受击效果)
bs.emitBGDynamics(position=m.pos,
                  chunkType='sweat',
                  velocity=m.forceDirection,
                  count=min(30, 1 + int(damage * 0.04)),
                  scale=0.9,
                  spread=0.28)

5.4. 全局变量 (bs.getSharedObject)
你可以获取和修改游戏引擎的全局属性，例如屏幕色调。
# 获取全局对象
g = bs.getSharedObject('globals')

# 修改屏幕色调
g.tint = (1.3, 1.2, 1.0) # (来自 Duel.py)

# 甚至可以通过 "connectAttr" 动画化色调
c = bs.newNode("combine", ...)
bs.animate(c, 'input0', {...}, loop=True)
c.connectAttr('output', bs.getSharedObject('globals'), 'tint')

6. 自定义对象 (e.g., bsBomb.Bomb)
与 bs.PlayerSpaz 类似，你可以继承其他 bs.Actor 子类，如 bsBomb.Bomb。
import bsBomb

class PowBox(bsBomb.Bomb):
    def __init__(self, position=(0, 1, 0), velocity=(0, 0, 0)):
        # 调用父类的 init，类型为 'tnt'
        bsBomb.Bomb.__init__(self, position, velocity,
                             bombType='tnt', blastRadius=2.5,
                             sourcePlayer=None, owner=None)
        self.setPowText() # 添加自定义文本

    # 重写 handleMessage 来改变行为
    def handleMessage(self, m):
        if isinstance(m, bs.PickedUpMessage):
            self._heldBy = m.node
        elif isinstance(m, bs.DroppedMessage):
            # 被丢下时，开始一个计时器准备爆炸
            bs.gameTimer(600, bs.WeakCall(self.pow))
        else:
            # 其他消息 (如 bs.HitMessage) 交给父类处理
            bsBomb.Bomb.handleMessage(self, m)

    def pow(self):
        self.explode() # 调用父类的爆炸方法
        
        
这是基本的api4的写法，欢迎大佬指出问题所在，包括可以上传更多的源码给我，我也会尽力来搞得
---
