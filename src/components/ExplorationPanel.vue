<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useGameStore } from '../stores/gameStore'
import { ElMessage } from 'element-plus'

const gameStore = useGameStore()

// 当前选中的区域
const selectedRegion = ref(null)

// 探索区域列表
const explorationRegions = computed(() => [
  {
    id: 'forest',
    name: '森林',
    description: '茂密的森林，蕴含丰富的木材和草药资源',
    difficulty: 1,
    unlockRequirements: { survival: 1 },
    resources: ['wood', 'herb', 'food'],
    dangers: ['predator', 'storm'],
    image: '🌲',
    explorationTime: 180, // 秒
    energyCost: 30,
    resourceCost: { food: 2, water: 2 }
  },
  {
    id: 'hills',
    name: '丘陵',
    description: '崎岖的丘陵地带，可以找到石头和少量金属',
    difficulty: 2,
    unlockRequirements: { survival: 2 },
    resources: ['stone', 'metal'],
    dangers: ['rockslide', 'predator'],
    image: '⛰️',
    explorationTime: 240,
    energyCost: 40,
    resourceCost: { food: 3, water: 3 }
  },
  {
    id: 'ruins',
    name: '废墟',
    description: '古老的废墟，可能藏有珍贵的科技碎片和遗物',
    difficulty: 3,
    unlockRequirements: { survival: 3, combat: 2 },
    resources: ['metal', 'parts', 'techFragment', 'ancientRelic'],
    dangers: ['rockslide', 'radiation', 'hostiles'],
    image: '🏚️',
    explorationTime: 360,
    energyCost: 50,
    resourceCost: { food: 5, water: 5, medicine: 1 }
  },
  {
    id: 'cave',
    name: '洞穴',
    description: '黑暗的洞穴系统，蕴含丰富的矿物资源',
    difficulty: 4,
    unlockRequirements: { survival: 4, combat: 3 },
    resources: ['stone', 'metal', 'crystal'],
    dangers: ['rockslide', 'thirst', 'creatures'],
    image: '🕳️',
    explorationTime: 420,
    energyCost: 60,
    resourceCost: { food: 6, water: 6, medicine: 2, tools: 1 }
  },
  {
    id: 'wasteland',
    name: '荒漠',
    description: '危险的辐射区域，但可能有高级科技残骸',
    difficulty: 5,
    unlockRequirements: { survival: 5, combat: 4 },
    resources: ['metal', 'parts', 'techFragment', 'ancientRelic', 'rareElement'],
    dangers: ['radiation', 'storm', 'hostiles', 'thirst'],
    image: '🏜️',
    explorationTime: 480,
    energyCost: 70,
    resourceCost: { food: 8, water: 10, medicine: 3, tools: 2 }
  }
])

// 可探索的区域（根据玩家技能等级过滤）
const availableRegions = computed(() => {
  return explorationRegions.value.filter(region => {
    // 检查技能要求
    for (const [skill, level] of Object.entries(region.unlockRequirements)) {
      if (gameStore.skills[skill] < level) return false
    }
    return true
  })
})

// 检查是否有足够的资源进行探索
const canExplore = computed(() => {
  if (!selectedRegion.value) return false
  const region = explorationRegions.value.find(r => r.id === selectedRegion.value)
  if (!region) return false
  // 检查体力
  if (gameStore.player.energy < region.energyCost) return false
  // 检查资源
  for (const [resource, amount] of Object.entries(region.resourceCost)) {
    if (gameStore.resources[resource] < amount) return false
  }
  return true
})

// 获取区域资源消耗文本
const getResourceCostText = (region) => {
  const costs = [`体力 ${region.energyCost}`]
  for (const [resource, amount] of Object.entries(region.resourceCost)) {
    costs.push(`${gameStore.getResourceName(resource)} ${amount}`)
  }
  return costs.join(', ')
}

// 获取区域可能发现的资源文本
const getPossibleResourcesText = (region) => {
  const costs = []
  for (const resource of region.resources) {
    costs.push(`${gameStore.getResourceName(resource)}`)
  }
  return costs.join(', ')
}

// 获取区域危险文本
const getDangersText = (region) => {
  const costs = []
  for (const resource of region.dangers) {
    costs.push(`${gameStore.getResourceName(resource)}`)
  }
  return costs.join(', ')
}

// 开始探索
const startExploration = () => {
  if (!selectedRegion.value || !canExplore.value) return
  const region = explorationRegions.value.find(r => r.id === selectedRegion.value)
  // 消耗体力
  gameStore.player.energy -= region.energyCost
  // 消耗资源
  for (const [resource, amount] of Object.entries(region.resourceCost)) {
    gameStore.consumeResource(resource, amount)
  }
  // 创建探索活动
  const explorationActivity = {
    id: `explore_${region.id}_${Date.now()}`,
    recipeId: `explore_${region.id}`,
    name: `探索${region.name}`,
    startTime: Date.now(),
    duration: region.explorationTime * 1000, // 转换为毫秒
    completed: false,
    region: region.id
  }
  // 添加到当前活动
  gameStore.currentActivities.push(explorationActivity)
  // 添加事件日志
  gameStore.addToEventLog(`开始探索${region.name}`)
  ElMessage.success(`开始探索${region.name}`)
  // 设置定时器完成探索
  setTimeout(() => completeExploration(explorationActivity.id, region), region.explorationTime * 1000)
  // 重置选中
  selectedRegion.value = null
}

// 完成探索
const completeExploration = (activityId, region) => {
  // 从当前活动中移除
  const activityIndex = gameStore.currentActivities.findIndex(a => a.id === activityId)
  if (activityIndex === -1) return
  gameStore.currentActivities.splice(activityIndex, 1)
  // 生成探索结果
  generateExplorationResults(region)
  // 增加相关技能经验
  gameStore.addSkillExp('survival', 2)
  if (region.difficulty >= 3) gameStore.addSkillExp('combat', 1)
}

// 生成探索结果
const generateExplorationResults = (region) => {
  // 基础发现率
  const baseDiscoveryChance = 0.7
  // 根据难度和技能调整发现率
  const survivalBonus = (gameStore.skills.survival - region.difficulty) * 0.05
  const discoveryChance = Math.min(0.95, Math.max(0.3, baseDiscoveryChance + survivalBonus))
  // 资源发现
  let resourcesFound = false
  for (const resource of region.resources) {
    // 根据资源稀有度调整发现概率
    let resourceChance = discoveryChance
    // 稀有资源发现率降低
    if (['techFragment', 'ancientRelic', 'rareElement'].includes(resource)) resourceChance *= 0.3
    if (Math.random() < resourceChance) {
      // 确定资源数量，基于难度和随机因素
      const baseAmount = region.difficulty * 2
      const randomFactor = Math.random() * 0.5 + 0.75 // 0.75 到 1.25 的随机因子
      const amount = Math.max(1, Math.floor(baseAmount * randomFactor))
      gameStore.addResource(resource, amount)
      gameStore.addToEventLog(`在${region.name}中发现了 ${gameStore.getResourceName(resource)}x${amount}`)
      resourcesFound = true
    }
  }
  if (!resourcesFound) gameStore.addToEventLog(`你在${region.name}中没有发现任何有价值的资源`)
  // 危险事件
  const dangerChance = 0.2 + (region.difficulty * 0.05) // 基础危险概率 + 难度加成
  if (Math.random() < dangerChance) {
    // 随机选择一个危险
    const danger = region.dangers[Math.floor(Math.random() * region.dangers.length)]
    handleDangerEvent(danger, region)
  }
  // 特殊发现（低概率）
  const specialDiscoveryChance = 0.05 + (gameStore.skills.survival * 0.01)
  if (Math.random() < specialDiscoveryChance) handleSpecialDiscovery(region)
}

// 处理危险事件
const handleDangerEvent = (danger, region) => {
  switch (danger) {
    case 'predator':
      if (gameStore.skills.combat >= region.difficulty) {
        gameStore.addToEventLog(`你在${region.name}遇到了野兽袭击，但成功击退了它`)
        gameStore.addResource('food', region.difficulty * 3)
        gameStore.addToEventLog('你获得了一些食物')
        gameStore.addSkillExp('combat', 2)
      } else {
        gameStore.player.health -= 10 * (region.difficulty - gameStore.skills.combat)
        gameStore.addToEventLog(`你在${region.name}遭遇野兽袭击，受了伤`)
      }
      break
    case 'storm':
      gameStore.player.energy -= 10
      gameStore.player.mental -= 5
      gameStore.addToEventLog(`你在${region.name}遭遇了风暴，消耗了额外的体力和精神`)
      break
    case 'rockslide':
      if (Math.random() < 0.5) {
        gameStore.player.health -= 15
        gameStore.addToEventLog(`你在${region.name}遭遇了坍塌，受了伤`)
      } else {
        gameStore.addToEventLog(`你在${region.name}险些遭遇坍塌，幸好及时躲避`)
      }
      break
    case 'radiation':
      if (gameStore.resources.medicine > 0) {
        gameStore.consumeResource('medicine', 1)
        gameStore.addToEventLog(`你在${region.name}受到了辐射，但使用药品进行了治疗`)
      } else {
        gameStore.player.health -= 20
        gameStore.addToEventLog(`你在${region.name}受到了辐射伤害，健康状况恶化`)
      }
      break
    case 'hostiles':
      if (gameStore.skills.combat > region.difficulty) {
        gameStore.addToEventLog(`你在${region.name}遇到了敌对人员，但成功击退了他们`)
        gameStore.addSkillExp('combat', 3)
      } else {
        gameStore.player.health -= 15
        // 随机失去一些资源
        const resourceTypes = ['food', 'water', 'medicine', 'tools']
        const lostResource = resourceTypes[Math.floor(Math.random() * resourceTypes.length)]
        const lostAmount = Math.min(gameStore.resources[lostResource], Math.floor(Math.random() * 5) + 1)
        gameStore.consumeResource(lostResource, lostAmount)
        gameStore.addToEventLog(`你在${region.name}遭遇敌对人员，受了伤并失去了一些${lostResource}`)
      }
      break
    case 'thirst':
      gameStore.consumeResource('water', 2)
      gameStore.addToEventLog(`你在${region.name}消耗了额外的水资源`)
      break
    case 'creatures':
      if (Math.random() < 0.7) {
        gameStore.player.mental -= 10
        gameStore.addToEventLog(`你在${region.name}遇到了奇怪的生物，感到恐惧`)
      } else {
        gameStore.addToEventLog(`你在${region.name}发现了奇怪的生物，但它们似乎对你不感兴趣`)
      }
      break
  }
  // 检查游戏结束条件
  if (gameStore.player.health <= 0) gameStore.gameOver()
}

// 处理特殊发现
const handleSpecialDiscovery = (region) => {
  const discoveries = [
    {
      name: '隐藏补给',
      effect: () => {
        gameStore.addResource('food', 10)
        gameStore.addResource('water', 10)
        gameStore.addResource('medicine', 2)
        gameStore.addToEventLog(`你在${region.name}发现了一个隐藏的补给缓存！`)
      },
      weight: 10
    },
    {
      name: '古代科技',
      effect: () => {
        gameStore.addResource('techFragment', 3)
        gameStore.addResource('parts', 2)
        gameStore.addToEventLog(`你在${region.name}发现了一些保存完好的古代科技！`)
      },
      weight: 5
    },
    {
      name: '幸存者',
      effect: () => {
        gameStore.addToEventLog(`你在${region.name}遇到了一位幸存者，他分享了一些生存知识`)
        gameStore.addSkillExp('survival', 3)
      },
      weight: 8
    },
    {
      name: '神秘遗物',
      effect: () => {
        gameStore.addResource('ancientRelic', 1)
        gameStore.addToEventLog(`你在${region.name}发现了一个神秘的古代遗物！`)
      },
      weight: 3
    },
    {
      name: '隐藏地图',
      effect: () => {
        // 解锁新区域的逻辑可以在这里实现
        gameStore.addToEventLog(`你在${region.name}发现了一张地图，标记了一些新的区域`)
        gameStore.addSkillExp('exploration', 2)
      },
      weight: 4
    }
  ]
  // 计算总权重
  const totalWeight = discoveries.reduce((sum, discovery) => sum + discovery.weight, 0)
  // 根据权重随机选择一个发现
  let randomWeight = Math.random() * totalWeight
  let selectedDiscovery = null
  for (const discovery of discoveries) {
    randomWeight -= discovery.weight
    if (randomWeight <= 0) {
      selectedDiscovery = discovery
      break
    }
  }
  if (selectedDiscovery) selectedDiscovery.effect()
}

// 活动进度和时间的响应式数据
const activityProgress = ref({})
const activityRemainingTime = ref({})
// 活动更新定时器
const activityTimerId = ref(null)

// 更新所有进行中活动的进度和时间
const updateActivitiesStatus = () => {
  gameStore.currentActivities.forEach(activity => {
    const now = Date.now()
    const elapsed = now - activity.startTime
    const progress = Math.min(100, (elapsed / activity.duration) * 100)
    activityProgress.value[activity.id] = progress
    const remaining = Math.max(0, activity.duration - elapsed)
    const seconds = Math.ceil(remaining / 1000)
    activityRemainingTime.value[activity.id] = seconds < 60 ? `${seconds}秒` : `${Math.floor(seconds / 60)}分${seconds % 60}秒`
  })
}

// 计算进行中的活动完成百分比
const getActivityProgress = (activity) => {
  if (activityProgress.value[activity.id] !== undefined) return activityProgress.value[activity.id]
  // 初始计算
  const now = Date.now()
  const elapsed = now - activity.startTime
  const progress = Math.min(100, (elapsed / activity.duration) * 100)
  return progress
}

// 计算活动剩余时间
const getActivityRemainingTime = (activity) => {
  if (activityRemainingTime.value[activity.id] !== undefined) return activityRemainingTime.value[activity.id]
  // 初始计算
  const now = Date.now()
  const elapsed = now - activity.startTime
  const remaining = Math.max(0, activity.duration - elapsed)
  const seconds = Math.ceil(remaining / 1000)
  if (seconds < 60) return `${seconds}秒`
  return `${Math.floor(seconds / 60)}分${seconds % 60}秒`
}

// 启动活动状态更新定时器
const startActivityTimer = () => {
  if (activityTimerId.value) return
  // 每秒更新一次活动状态
  activityTimerId.value = setInterval(() => {
    if (gameStore.gameState === 'playing' && gameStore.currentActivities.length > 0) updateActivitiesStatus()
  }, 1000)
}

// 组件挂载时启动定时器
onMounted(() => startActivityTimer())

// 组件卸载时清除定时器
onUnmounted(() => {
  if (activityTimerId.value) {
    clearInterval(activityTimerId.value)
    activityTimerId.value = null
  }
})
</script>

<template>
  <el-scrollbar class="exploration-panel">
    <div class="current-explorations" v-if="gameStore.currentActivities.some(a => a.recipeId.startsWith('explore_'))">
      <h4>进行中的探索</h4>
      <div class="exploration-list">
        <div v-for="activity in gameStore.currentActivities.filter(a => a.recipeId.startsWith('explore_'))"
          :key="activity.id" class="exploration-card in-progress">
          <div class="exploration-header">
            <div class="exploration-name">{{ activity.name }}</div>
            <div class="exploration-time">
              剩余: {{ getActivityRemainingTime(activity) }}
            </div>
          </div>
          <el-progress :percentage="getActivityProgress(activity)" :stroke-width="10" :show-text="false" />
        </div>
      </div>
    </div>
    <div class="available-regions">
      <h4>可探索区域</h4>
      <div class="region-list">
        <div v-for="region in availableRegions" :key="region.id" class="region-card"
          :class="{ 'selected': selectedRegion === region.id }" @click="selectedRegion = region.id">
          <div class="region-header">
            <div class="region-icon">{{ region.image }}</div>
            <div class="region-name">{{ region.name }}</div>
            <div class="region-difficulty">
              难度: {{ '⭐'.repeat(region.difficulty) }}
            </div>
          </div>
          <div class="region-description">{{ region.description }}</div>
          <div class="region-details">
            <div class="region-resources">
              <div>可能发现: {{ getPossibleResourcesText(region) }}</div>
              <div>潜在危险: {{ getDangersText(region) }}</div>
            </div>
            <div class="region-requirements">
              需要: {{ getResourceCostText(region) }}
            </div>
          </div>
        </div>
        <div v-if="availableRegions.length === 0" class="no-regions-message">
          当前没有可探索的区域，提升你的生存和战斗技能以解锁更多区域
        </div>
      </div>
    </div>
    <div v-if="selectedRegion" class="exploration-actions">
      <el-button type="primary" @click="startExploration" :disabled="!canExplore">
        {{ canExplore ? '开始探索' : '资源不足' }}
      </el-button>
    </div>
  </el-scrollbar>
</template>

<style scoped>
.exploration-panel {
  background-color: var(--el-bg-color-overlay);
  border-radius: 4px;
  padding: 15px;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.exploration-panel h3 {
  margin-top: 0;
  margin-bottom: 15px;
}

.current-explorations {
  margin-bottom: 20px;
}

.exploration-list,
.region-list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 15px;
  margin-bottom: 20px;
}

.exploration-card,
.region-card {
  background-color: var(--el-bg-color);
  border-radius: 4px;
  padding: 12px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  transition: transform 0.3s;
}

.region-card {
  cursor: pointer;
  border-left: 4px solid #909399;
}

.region-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 4px 16px 0 rgba(0, 0, 0, 0.2);
}

.region-card.selected {
  border-left: 4px solid #409EFF;
  background-color: var(--el-color-primary-light-9);
}

.exploration-card.in-progress {
  border-left: 4px solid #E6A23C;
}

.region-header {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.region-icon {
  font-size: 1.5em;
  margin-right: 10px;
}

.region-name {
  font-weight: bold;
  flex: 1;
}

.region-difficulty {
  font-size: 0.8em;
  color: #E6A23C;
}

.region-description {
  font-size: 0.9em;
  color: var(--el-text-color-secondary);
  margin-bottom: 10px;
}

.region-details {
  font-size: 0.85em;
  color: var(--el-text-color-secondary);
}

.region-resources {
  margin-bottom: 5px;
}

.region-requirements {
  margin-top: 8px;
  font-weight: bold;
}

.exploration-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.exploration-name {
  font-weight: bold;
}

.exploration-time {
  font-size: 0.8em;
  color: var(--el-text-color-secondary);
}

.exploration-actions {
  margin-top: 15px;
  display: flex;
  justify-content: center;
}

.no-regions-message {
  grid-column: 1 / -1;
  padding: 15px;
  text-align: center;
  color: var(--el-text-color-secondary);
  font-style: italic;
}

.available-regions {
  flex: 1;
  overflow-y: auto;
}

@media (max-width: 768px) {

  .exploration-list,
  .region-list {
    grid-template-columns: 1fr;
  }
}
</style>