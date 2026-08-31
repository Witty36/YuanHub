<template>
  <section class="growth-tracker" aria-labelledby="growth-tracker-title">
    <div class="tracker-heading">
      <div>
        <span class="section-kicker">特别关注 · 养成追踪</span>
        <h2 id="growth-tracker-title">把下一份资源用在想练的人身上</h2>
        <p>等级、修为、化极的固定需求来自 Wiki 计算规则；五铢钱只展示总量，不参与 ETA。</p>
      </div>
      <button class="tracker-refresh" type="button" :disabled="loading || !accountId" title="刷新库存和流水" @click="loadInventory(true)">
        <RefreshCw :size="16" :class="{ spin: loading }" aria-hidden="true" />
        <span>{{ loading ? '同步中' : '刷新数据' }}</span>
      </button>
    </div>

    <div class="tracker-mode-switch" role="group" aria-label="养成追踪视图">
      <button type="button" :aria-pressed="trackerMode === 'favorite'" :class="{ on: trackerMode === 'favorite' }" @click="trackerMode = 'favorite'">特别关注</button>
      <button type="button" :aria-pressed="trackerMode === 'growth'" :class="{ on: trackerMode === 'growth' }" @click="trackerMode = 'growth'">练度</button>
      <button type="button" :aria-pressed="trackerMode === 'heart'" :class="{ on: trackerMode === 'heart' }" @click="trackerMode = 'heart'">心纸</button>
    </div>

    <div v-if="!isLoggedIn || !accountId" class="tracker-state">
      <Info :size="18" aria-hidden="true" />
      <span>登录并选择子账号后，才能读取库存、流水和特别关注名单。</span>
    </div>
    <div v-else-if="error" class="tracker-state is-error" role="alert">
      <Info :size="18" aria-hidden="true" />
      <span>{{ error }}</span>
      <button type="button" @click="loadInventory(true)">重试</button>
    </div>
    <template v-else>
      <template v-if="trackerMode === 'favorite'">
      <div class="tracker-overview" aria-label="养成追踪概览">
        <div class="overview-cell"><span>特别关注</span><strong>{{ favoriteRows.length }}<small>位</small></strong><em>{{ ownedFavoriteCount }} 位已招募</em></div>
        <div class="overview-cell"><span>心纸库存</span><strong>{{ formatNumber(totalHeartStock) }}<small>张</small></strong><em>关注密探合计</em></div>
        <div class="overview-cell"><span>本期获得</span><strong>{{ formatNumber(totalHeartAcquired) }}<small>张</small></strong><em>近 {{ rangeDays }} 日</em></div>
        <div class="overview-cell"><span>待补项</span><strong>{{ totalGapCount }}<small>项</small></strong><em>按密探目标合计</em></div>
      </div>

      <div v-if="!favoriteRows.length" class="tracker-state empty">
        <Star :size="18" aria-hidden="true" />
        <span>还没有特别关注的密探。先在图鉴卡片右上角点亮星标。</span>
      </div>

      <template v-else>
        <section class="aggregate-plan" aria-labelledby="aggregate-plan-title">
          <div class="aggregate-head">
            <div><h3 id="aggregate-plan-title">关注目标总账</h3><p>所有关注目标合并计算，共享库存只抵扣一次。</p></div>
            <span>五铢钱 {{ formatMoney(aggregatePlan.total.money) }}</span>
          </div>
          <div v-if="!aggregatePlan.materialGaps.length && !aggregatePlan.heartGap" class="materials-clear">全部目标资源已备齐</div>
          <div v-else class="material-chips">
            <span v-for="gap in aggregatePlan.materialGaps" :key="gap.id" class="material-chip">
              <b>{{ itemName(gap.id) }}</b><em>缺 {{ formatNumber(gap.gap) }}</em><small>{{ rateLabel(gap.id) }}</small>
            </span>
            <span v-if="aggregatePlan.heartGap" class="material-chip heart-chip">
              <b>心纸</b><em>共缺 {{ formatNumber(aggregatePlan.heartGap) }}</em><small>按密探分别计算</small>
            </span>
          </div>
          <p class="aggregate-eta">{{ aggregateEtaLabel }}</p>
        </section>

        <p v-if="targetError" class="tracker-target-error" role="alert">{{ targetError }}</p>
        <p v-else-if="targetNotice" class="tracker-target-notice" role="status">{{ targetNotice }}</p>
        <div class="tracker-list">
        <article v-for="row in favoriteRows" :key="row.id" class="tracker-row">
          <div class="tracker-row-head">
            <div class="tracker-identity">
              <div class="tracker-avatar" :class="'rarity-r' + (row.rarity || 3)">
                <img v-if="row.avatar" :src="avatarUrl(row.avatar)" :alt="row.name" loading="lazy" />
                <span v-else>{{ monogram(row) }}</span>
              </div>
              <div>
                <h3>{{ row.name || row.id }}</h3>
                <p>
                  <span class="tracker-prof"><img v-if="profIcon(row.prof)" :src="profIcon(row.prof)" alt="" aria-hidden="true" />{{ row.prof || '未知属性' }}</span>
                  <span>{{ firstSubProf(row) || '未标注职业' }}</span>
                  <span>{{ row.owned ? '已拥有' : '未拥有' }}</span>
                </p>
              </div>
            </div>
            <div class="tracker-targets" aria-label="设置养成目标">
              <label>目标等级 <input type="number" :min="row.level" max="100" :value="targetFor(row).level" :disabled="targetLoading || targetBusyIds.has(row.id)" @change="setTarget(row, 'level', $event)" /></label>
              <label>目标修为 <input type="number" :min="row.elite" :max="maxEliteForLevel(row.level)" :value="targetFor(row).elite" :disabled="targetLoading || targetBusyIds.has(row.id)" @change="setTarget(row, 'elite', $event)" /></label>
              <label>目标节点
                <select :value="targetFor(row).starLevel" :disabled="targetLoading || targetBusyIds.has(row.id)" @change="setTarget(row, 'starLevel', $event)">
                  <option v-for="stage in starStagesFor(row)" :key="stage.value" :value="stage.value">{{ stage.label }}</option>
                </select>
              </label>
              <label>目标心纸 <input type="number" min="0" max="1000000" :value="targetFor(row).heartPaper > 0 ? targetFor(row).heartPaper : ''" placeholder="随化极" :disabled="targetLoading || targetBusyIds.has(row.id)" @change="setTarget(row, 'heartPaper', $event)" /></label>
            </div>
          </div>

          <div class="tracker-progress-grid">
            <div class="progress-block">
              <div class="progress-title"><span>等级</span><b>Lv{{ row.level }} / {{ targetFor(row).level }}</b></div>
              <div class="progress-track"><i :style="{ width: progress(row.level, targetFor(row).level) + '%' }"></i></div>
              <p>{{ formatNumber(row.calculation.level.experience) }} 经验 · 兵书残卷约 {{ formatNumber(row.calculation.level.books.fragment) }} 卷</p>
            </div>
            <div class="progress-block">
              <div class="progress-title"><span>修为</span><b>{{ row.elite }} / {{ targetFor(row).elite }}</b></div>
              <div class="progress-track mint"><i :style="{ width: progress(row.elite, targetFor(row).elite) + '%' }"></i></div>
              <p>{{ materialSummary(row.calculation.xiuwei) || '无需补充修为材料' }}</p>
            </div>
            <div class="progress-block">
              <div class="progress-title"><span>化极</span><b>{{ starLabel(row.starLevel) }} / {{ starLabel(targetFor(row).starLevel) }}</b></div>
              <div class="progress-track rose"><i :style="{ width: progress(starStage(row.starLevel), starStage(targetFor(row).starLevel)) + '%' }"></i></div>
              <p>持有 {{ formatNumber(row.calculation.heartOwned) }} · 需 {{ formatNumber(row.calculation.heartRequired) }} · 缺 <strong>{{ formatNumber(row.calculation.heartGap) }}</strong> · 本期 +{{ formatNumber(row.heartAcquired) }}</p>
            </div>
          </div>

          <div class="tracker-materials">
            <div class="materials-head"><span>单人缺口</span><small>近 {{ rangeDays }} 日流水 · 日均按自然日计算</small></div>
            <div v-if="!row.calculation.gaps.length && !row.calculation.heartGap" class="materials-clear">当前目标材料已备齐</div>
            <div v-else class="material-chips">
              <span v-for="gap in row.calculation.gaps" :key="gap.id" class="material-chip">
                <b>{{ itemName(gap.id) }}</b><em>缺 {{ formatNumber(gap.gap) }}</em><small>{{ rateLabel(gap.id) }}</small>
              </span>
              <span v-if="row.calculation.heartGap" class="material-chip heart-chip">
                <b>心纸</b><em>缺 {{ formatNumber(row.calculation.heartGap) }}</em><small>{{ heartRateLabel(row.id) }}</small>
              </span>
            </div>
            <div class="eta-line">
              <span v-if="row.calculation.etaDays != null">按当前速度，最慢材料约 {{ formatEta(row.calculation.etaDays) }}</span>
              <span v-else>暂无 ETA：没有足够的对应材料流水</span>
              <span class="money-total">五铢钱需求 {{ formatMoney(row.calculation.total.money) }}</span>
            </div>
          </div>
        </article>
        </div>
        </template>
        </template>
      <template v-else-if="trackerMode === 'growth'">
        <div class="tracker-mode-body">
          <div class="tracker-mode-summary">
            <strong>{{ growthRows.length }}</strong> 位密探 · 练度缺口汇总
            <div class="tracker-mode-actions">
              <button type="button" class="tracker-edit-button" :class="{ on: growthAllocationPanel }" @click="growthAllocationPanel = !growthAllocationPanel"><span>库存分配方式</span></button>
              <button type="button" class="tracker-edit-button" :class="{ on: viewEditor === 'growth' }" @click="toggleViewEditor('growth')"><Pencil :size="14" aria-hidden="true" /><span>编辑目标</span></button>
            </div>
          </div>
          <p v-if="targetError" class="tracker-target-error" role="alert">{{ targetError }}</p>
          <p v-else-if="targetNotice" class="tracker-target-notice" role="status">{{ targetNotice }}</p>
          <div v-if="growthAllocationPanel" class="growth-allocation-panel">
            <div class="growth-allocation-head">
              <div class="allocation-mode" role="group" aria-label="库存分配方式">
                <button type="button" :aria-pressed="growthAllocationMode === 'priority'" :class="{ on: growthAllocationMode === 'priority' }" @click="setGrowthAllocationMode('priority')">按优先级</button>
                <button type="button" :aria-pressed="growthAllocationMode === 'ratio'" :class="{ on: growthAllocationMode === 'ratio' }" @click="setGrowthAllocationMode('ratio')">按比例</button>
              </div>
              <button v-if="growthAllocationMode === 'priority'" type="button" class="allocation-reset" :disabled="!growthPriorityIds.length" @click="resetGrowthPriority">恢复自动顺序</button>
            </div>
            <p v-if="growthAllocationMode === 'priority'" class="allocation-hint">按优先级：按顺序逐个分配库存，靠前的密探优先毕业；默认绝密 &gt; 机密 &gt; 隐密、耗材多的优先，可拖拽调整顺序。</p>
            <p v-else class="allocation-hint">按比例：把库存按每个密探需求的占比平均分摊，所有密探都会分到一部分，不涉及优先级排序。</p>
            <div v-if="growthAllocationMode === 'priority'" class="growth-allocation-list">
              <div v-for="(row, index) in growthAllocationOrderRows" :key="row.id" class="growth-allocation-row" :class="{ ready: row.allocationReady, dragging: growthDragIndex === index }" draggable="true" @dragstart="startGrowthDrag(index)" @dragend="endGrowthDrag" @dragover.prevent @drop.prevent="dropGrowthPriority(index)">
                <span class="allocation-rank">{{ index + 1 }}</span>
                <span class="allocation-drag" aria-hidden="true">⠿</span>
                <span class="allocation-name">{{ row.name || row.id }}</span>
                <span class="allocation-rarity"><span v-if="!row.owned" class="allocation-unowned">未拥有</span>{{ growthRarityLabel(row.rarity) }}</span>
                <span class="allocation-status">{{ row.allocationReady ? '可升级' : '缺材料' }}</span>
                <span class="allocation-actions">
                  <button type="button" :aria-label="'上移 ' + (row.name || row.id)" :disabled="index === 0" @click="moveGrowthPriority(row.id, -1)">↑</button>
                  <button type="button" :aria-label="'下移 ' + (row.name || row.id)" :disabled="index === growthAllocationOrderRows.length - 1" @click="moveGrowthPriority(row.id, 1)">↓</button>
                </span>
              </div>
            </div>
          </div>
          <div v-if="viewEditor === 'growth'" class="tracker-view-editor">
            <div class="tracker-view-search">
              <input v-model.trim="viewSearch" type="search" placeholder="搜索名称 / 拼音 / 首字母" aria-label="搜索密探" />
              <button type="button" class="view-search-clear" aria-label="清空搜索" title="清空搜索" @click="viewSearch = ''">×</button>
              <label class="tracker-owned-only" :class="{ on: viewOwnedOnly }"><input v-model="viewOwnedOnly" type="checkbox" /><span class="tracker-owned-check"><i></i></span><span>仅显示已拥有密探</span></label>
            </div>
            <div class="tracker-view-filters">
              <div class="pf-row">
                <span class="pf-label">属性</span>
                <div class="mf-filter" role="group" aria-label="按属性筛选密探">
                  <button type="button" :aria-pressed="viewProfFilter === 'all'" :class="{ on: viewProfFilter === 'all' }" @click="viewProfFilter = 'all'">全部</button>
                  <button v-for="p in viewProfOptions" :key="p" type="button" :aria-pressed="viewProfFilter === p" :class="{ on: viewProfFilter === p }" @click="viewProfFilter = p"><img v-if="profIcon(p)" :src="profIcon(p)" alt="" aria-hidden="true" />{{ p }}</button>
                </div>
              </div>
              <div v-if="viewSubProfOptions.length" class="pf-row">
                <span class="pf-label">职业</span>
                <div class="mf-filter" role="group" aria-label="按职业筛选密探">
                  <button type="button" :aria-pressed="viewSubProfFilter === 'all'" :class="{ on: viewSubProfFilter === 'all' }" @click="viewSubProfFilter = 'all'">全部</button>
                  <button v-for="s in viewSubProfOptions" :key="s" type="button" :aria-pressed="viewSubProfFilter === s" :class="{ on: viewSubProfFilter === s }" @click="viewSubProfFilter = s">{{ s }}</button>
                </div>
              </div>
            </div>
            <div class="tracker-view-results">
              <button v-for="op in viewSearchOptions" :key="op.id" type="button" class="tracker-view-result" :class="[{ on: isViewSelected(op.id) }, 'rarity-r' + (op.rarity || 3)]" @click="selectViewOperator(op)">
                <span class="tracker-avatar">
                  <img v-if="op.avatar" :src="avatarUrl(op.avatar)" :alt="op.name" loading="lazy" />
                  <span v-else>{{ monogram(op) }}</span>
                </span>
                <b>{{ op.name || op.id }}</b>
                <small>{{ entryMeta(op) }}</small>
                <i v-if="isViewSelected(op.id)" aria-hidden="true">✓</i>
              </button>
              <p v-if="!viewSearchOptions.length" class="tracker-view-empty">没有匹配的密探</p>
            </div>
            <div v-if="selectedViewOperators.length" class="tracker-view-selected">
              <div class="tracker-view-selected-head">
                <strong>已选 {{ selectedViewOperators.length }} 位密探</strong>
                <button type="button" class="tracker-view-clear" @click="clearViewSelected">清空</button>
              </div>
              <div v-for="op in selectedViewOperators" :key="op.id" class="tracker-view-edit-row">
                <div class="tracker-view-edit-head">
                  <div class="tracker-avatar" :class="'rarity-r' + (op.rarity || 3)">
                    <img v-if="op.avatar" :src="avatarUrl(op.avatar)" :alt="op.name" loading="lazy" />
                    <span v-else>{{ monogram(op) }}</span>
                  </div>
                  <div class="tracker-view-selected-name">
                    <h3>{{ op.name || op.id }}</h3>
                    <p><span class="tracker-prof"><img v-if="profIcon(op.prof)" :src="profIcon(op.prof)" alt="" aria-hidden="true" />{{ op.prof || '未知属性' }}</span><span>{{ firstSubProf(op) || '未标注职业' }}</span><span v-if="!entryOwned(op)" class="selected-unowned">未拥有</span><span v-if="entryOwned(op)">当前 等级{{ currentFor(op).level }} · 修为{{ currentFor(op).elite }}</span></p>
                  </div>
                </div>
                <label>目标等级 <input v-model.number="viewDraft[op.id].level" type="number" min="1" max="100" @input="markViewChanged(op.id)" /></label>
                <label>目标修为 <input v-model.number="viewDraft[op.id].elite" type="number" min="0" max="17" @input="markViewChanged(op.id)" /></label>
                <div class="tracker-view-row-actions">
                  <button type="button" class="row-delete" :disabled="viewSaving || viewSavedIds.has(op.id)" @click="deleteViewRow(op.id)">删除</button>
                </div>
              </div>
            </div>
            <p v-else class="editor-groups-hint">还没有选择密探，从上方筛选面板点击密探加入。</p>
            <div class="tracker-view-actions">
              <button type="button" class="tracker-view-save-all" :disabled="viewSaving" @click="saveViewTarget('growth')">{{ viewSaving ? '保存中…' : '编辑完成（' + selectedViewOperators.length + '）' }}</button>
              <button type="button" class="cancel" :disabled="viewSaving" @click="closeViewEditor">取消编辑</button>
            </div>
          </div>
          <section class="aggregate-plan">
            <div class="aggregate-head">
              <div><h3>练度目标总账</h3><p>已设置练度目标的密探，缺口按当前库存抵扣。</p></div>
              <span>五铢钱 {{ formatMoney(growthAggregate.total.money) }}</span>
            </div>
            <div v-if="!growthAggregate.gapCount" class="materials-clear">当前练度目标材料已备齐</div>
            <div v-else class="material-chips">
              <span v-for="gap in growthAggregate.gaps" :key="gap.id" class="material-chip">
                <b>{{ itemName(gap.id) }}</b><em>缺 {{ formatNumber(gap.gap) }}</em><small>{{ rateLabel(gap.id) }}</small>
              </span>
            </div>
            <p class="aggregate-eta">{{ growthAggregateEtaLabel }}</p>
          </section>
          <div v-if="!growthRows.length" class="tracker-state empty">还没有设置练度目标的密探，点“编辑目标”添加。</div>
          <div v-else class="growth-compact-list">
            <article v-for="row in growthRows" :key="row.id" class="growth-compact-row">
              <header class="growth-compact-head">
                <div class="tracker-avatar" :class="'rarity-r' + (row.rarity || 3)">
                  <img v-if="row.avatar" :src="avatarUrl(row.avatar)" :alt="row.name" loading="lazy" />
                  <span v-else>{{ monogram(row) }}</span>
                </div>
                <div class="growth-compact-title">
                  <h3>{{ row.name || row.id }}</h3>
                  <p class="growth-compact-meta"><img v-if="profIcon(row.prof)" :src="profIcon(row.prof)" alt="" aria-hidden="true" />{{ row.prof || '未知属性' }} · {{ firstSubProf(row) || '未标注职业' }}</p>
                </div>
              </header>
              <div class="growth-compare">
                <div class="growth-compare-row head pair-row"><span></span><b>当前</b><b></b><b>目标</b></div>
                <div class="growth-compare-row pair-row"><span>等级</span><b>{{ row.level }}</b><i>/</i><b>{{ row.targetLevel }}</b></div>
                <div class="growth-compare-bar" :class="{ done: Number(row.level) >= Number(row.targetLevel) }"><i :style="{ width: progress(row.level, row.targetLevel) + '%' }"></i></div>
                <div class="growth-compare-row pair-row"><span>修为</span><b>{{ row.elite }}</b><i>/</i><b>{{ row.targetElite }}</b></div>
                <div class="growth-compare-bar" :class="{ done: Number(row.elite) >= Number(row.targetElite) }"><i :style="{ width: progress(row.elite, row.targetElite) + '%' }"></i></div>
                <div class="growth-compare-gap">
                  <span>缺口</span>
                  <b>{{ growthTargetReached(row) ? '无' : gapSummary(row) }}</b>
                </div>
                <div class="growth-upgrade-bar" :class="{ 'is-complete': growthTargetReached(row) }">
                  <em v-if="!row.owned" class="growth-gap-ready is-unowned">未拥有</em>
                  <template v-else-if="growthUpgradeFields(row).length">
                    <button v-if="growthHasUpgrade(row, 'level')" type="button" class="growth-upgrade-button" :class="{ ready: growthUpgradeReady(row, 'level') }" :disabled="growthUpgradeExecuting" @click="openGrowthUpgrade(row, 'level')">升 {{ growthUpgradeStepDelta(row, 'level') }} 级</button>
                    <button v-if="growthHasUpgrade(row, 'elite')" type="button" class="growth-upgrade-button" :class="{ ready: growthUpgradeReady(row, 'elite') }" :disabled="growthUpgradeExecuting" @click="openGrowthUpgrade(row, 'elite')">升至 {{ growthStepTarget(row, 'elite') }} 修为</button>
                  </template>
                  <em v-else-if="growthTargetReached(row)" class="growth-gap-ready">已备齐</em>
                </div>
                <div v-if="growthUpgradeKey === row.id" class="growth-upgrade-pop" role="dialog" aria-label="升级确认">
                  <div class="growth-upgrade-pop-head">
                    <strong>{{ growthUpgradeReady(row, growthUpgradeField) ? '升级' : '缺口检查' }} {{ row.name || row.id }} · {{ growthFieldLabel(growthUpgradeField) }}</strong>
                    <button type="button" aria-label="关闭" :disabled="growthUpgradeExecuting" @click="closeGrowthUpgrade">×</button>
                  </div>
                  <div class="growth-upgrade-pop-row">
                    <div class="growth-upgrade-pop-title"><b>{{ growthFieldLabel(growthUpgradeField) }}</b><span>{{ row[growthUpgradeField] }} → {{ growthStepTarget(row, growthUpgradeField) }}</span></div>
                    <div v-if="growthUpgradePreview(row, growthUpgradeField)" class="growth-upgrade-pop-body">
                      <div v-if="growthUpgradeRequirements(row, growthUpgradeField).length" class="growth-upgrade-pop-mats">
                        <span v-for="req in growthUpgradeRequirements(row, growthUpgradeField)" :key="(req.entity_type || req.entityType || 'item') + ':' + req.id" :class="{ 'is-lack': growthUpgradeReqValue(req, 'balance_after') < 0 }">
                          {{ itemName(req.id) }} × {{ formatNumber(growthUpgradeReqValue(req, 'required')) }}<small>{{ growthUpgradeReqLabel(req) }}</small>
                        </span>
                      </div>
                      <em v-else>无需扣除道具</em>
                      <em v-for="reason in growthUpgradeBlocking(row, growthUpgradeField)" :key="reason.code || reason.message" class="growth-upgrade-blocked">{{ reason.message || reason.code }}</em>
                    </div>
                    <div v-else-if="growthUpgradeError(row, growthUpgradeField)" class="growth-upgrade-pop-error">{{ growthUpgradeError(row, growthUpgradeField) }}</div>
                    <div v-else class="growth-upgrade-pop-state">正在核对库存…</div>
                  </div>
                  <em v-if="!growthUpgradeReady(row, growthUpgradeField)" class="growth-upgrade-pop-note">材料尚未备齐，当前仅可查看</em>
                  <div class="growth-upgrade-pop-actions">
                    <button type="button" class="growth-upgrade-pop-confirm" :disabled="!growthUpgradeCanConfirm(row)" @click="executeGrowthUpgrade(row)">{{ growthUpgradeExecuting ? '正在升级…' : '确认升级并扣除材料' }}</button>
                    <button type="button" class="growth-upgrade-pop-cancel" :disabled="growthUpgradeExecuting" @click="closeGrowthUpgrade">取消</button>
                  </div>
                </div>
              </div>
            </article>
          </div>
        </div>
      </template>
      <template v-else>
        <div class="tracker-mode-body">
          <div class="tracker-mode-summary">
            <strong>{{ heartRows.length }}</strong> 位密探 · 心纸缺口汇总
            <div class="tracker-mode-actions">
              <button type="button" class="tracker-edit-button" :class="{ on: viewEditor === 'heart' }" @click="toggleViewEditor('heart')"><Pencil :size="14" aria-hidden="true" /><span>编辑目标</span></button>
            </div>
          </div>
          <p v-if="targetError" class="tracker-target-error" role="alert">{{ targetError }}</p>
          <p v-else-if="targetNotice" class="tracker-target-notice" role="status">{{ targetNotice }}</p>
          <div v-if="viewEditor === 'heart'" class="tracker-view-editor">
            <div class="tracker-view-search">
              <input v-model.trim="viewSearch" type="search" placeholder="搜索名称 / 拼音 / 首字母" aria-label="搜索密探" />
              <button type="button" class="view-search-clear" aria-label="清空搜索" title="清空搜索" @click="viewSearch = ''">×</button>
              <label class="tracker-owned-only" :class="{ on: viewOwnedOnly }"><input v-model="viewOwnedOnly" type="checkbox" /><span class="tracker-owned-check"><i></i></span><span>仅显示已拥有密探</span></label>
            </div>
            <div class="tracker-view-filters">
              <div class="pf-row">
                <span class="pf-label">属性</span>
                <div class="mf-filter" role="group" aria-label="按属性筛选密探">
                  <button type="button" :aria-pressed="viewProfFilter === 'all'" :class="{ on: viewProfFilter === 'all' }" @click="viewProfFilter = 'all'">全部</button>
                  <button v-for="p in viewProfOptions" :key="p" type="button" :aria-pressed="viewProfFilter === p" :class="{ on: viewProfFilter === p }" @click="viewProfFilter = p"><img v-if="profIcon(p)" :src="profIcon(p)" alt="" aria-hidden="true" />{{ p }}</button>
                </div>
              </div>
              <div v-if="viewSubProfOptions.length" class="pf-row">
                <span class="pf-label">职业</span>
                <div class="mf-filter" role="group" aria-label="按职业筛选密探">
                  <button type="button" :aria-pressed="viewSubProfFilter === 'all'" :class="{ on: viewSubProfFilter === 'all' }" @click="viewSubProfFilter = 'all'">全部</button>
                  <button v-for="s in viewSubProfOptions" :key="s" type="button" :aria-pressed="viewSubProfFilter === s" :class="{ on: viewSubProfFilter === s }" @click="viewSubProfFilter = s">{{ s }}</button>
                </div>
              </div>
            </div>
            <div class="tracker-view-results">
              <button v-for="op in viewSearchOptions" :key="op.id" type="button" class="tracker-view-result" :class="[{ on: isViewSelected(op.id) }, 'rarity-r' + (op.rarity || 3)]" @click="selectViewOperator(op)">
                <span class="tracker-avatar">
                  <img v-if="op.avatar" :src="avatarUrl(op.avatar)" :alt="op.name" loading="lazy" />
                  <span v-else>{{ monogram(op) }}</span>
                </span>
                <b>{{ op.name || op.id }}</b>
                <small>{{ entryMeta(op) }}</small>
                <i v-if="isViewSelected(op.id)" aria-hidden="true">✓</i>
              </button>
              <p v-if="!viewSearchOptions.length" class="tracker-view-empty">没有匹配的密探</p>
            </div>
            <div v-if="selectedViewOperators.length" class="tracker-view-selected">
              <div class="tracker-view-selected-head">
                <strong>已选 {{ selectedViewOperators.length }} 位密探</strong>
                <button type="button" class="tracker-view-clear" @click="clearViewSelected">清空</button>
              </div>
              <div v-for="op in selectedViewOperators" :key="op.id" class="tracker-view-edit-row">
                <div class="tracker-view-edit-head">
                  <div class="tracker-avatar" :class="'rarity-r' + (op.rarity || 3)">
                    <img v-if="op.avatar" :src="avatarUrl(op.avatar)" :alt="op.name" loading="lazy" />
                    <span v-else>{{ monogram(op) }}</span>
                  </div>
                  <div class="tracker-view-selected-name">
                    <h3>{{ op.name || op.id }}</h3>
                    <p><span class="tracker-prof"><img v-if="profIcon(op.prof)" :src="profIcon(op.prof)" alt="" aria-hidden="true" />{{ op.prof || '未知属性' }}</span><span>{{ firstSubProf(op) || '未标注职业' }}</span><span v-if="!entryOwned(op)" class="selected-unowned">未拥有</span></p>
                  </div>
                </div>
                <label><span class="tracker-current-star">当前化极 <b>{{ starLabel(currentFor(op).starLevel) }}</b></span> → 目标化极
                  <select v-model.number="viewDraft[op.id].starLevel" @change="markViewChanged(op.id)">
                    <option v-for="stage in starStages" :key="stage.value" :value="stage.value">{{ stage.label }}</option>
                  </select>
                </label>
                <div class="tracker-view-row-actions">
                  <button type="button" class="row-delete" :disabled="viewSaving || viewSavedIds.has(op.id)" @click="deleteViewRow(op.id)">删除</button>
                </div>
              </div>
            </div>
            <div class="tracker-view-actions">
              <button type="button" class="tracker-view-save-all" :disabled="viewSaving" @click="saveViewTarget('heart')">{{ viewSaving ? '保存中…' : '编辑完成（' + selectedViewOperators.length + '）' }}</button>
              <button type="button" class="cancel" :disabled="viewSaving" @click="closeViewEditor">取消编辑</button>
            </div>
          </div>
          <section class="aggregate-plan">
            <div class="aggregate-head">
              <div><h3>心纸目标总账</h3><p>已设置化极/心纸目标的密探，按目标节点计算缺口。</p></div>
              <span>共缺 {{ formatNumber(heartAggregate.gap) }} 张</span>
            </div>
            <div v-if="heartAggregate.gap <= 0" class="materials-clear">当前心纸目标已备齐</div>
            <div v-else class="material-chips">
              <span class="material-chip heart-chip">
                <b>心纸</b><em>缺 {{ formatNumber(heartAggregate.gap) }}</em><small>持有 {{ formatNumber(heartAggregate.stock) }} · 目标 {{ formatNumber(heartAggregate.required) }}</small>
              </span>
            </div>
            <p class="aggregate-eta">{{ heartAggregateEtaLabel }}</p>
          </section>
          <div v-if="!heartRows.length" class="tracker-state empty">还没有设置化极/心纸目标的密探，点“编辑目标”添加。</div>
          <div v-else class="heart-grid">
            <article v-for="row in heartRows" :key="row.id" class="heart-cell">
              <div class="heart-cell-head">
                <div class="tracker-avatar" :class="'rarity-r' + (row.rarity || 3)">
                  <img v-if="row.avatar" :src="avatarUrl(row.avatar)" :alt="row.name" loading="lazy" />
                  <span v-else>{{ monogram(row) }}</span>
                </div>
                <div>
                  <h3>{{ row.name || row.id }}</h3>
                  <p>{{ starLabel(row.currentStar) }} → {{ row.targetStarLabel }}</p>
                </div>
              </div>
              <div class="heart-cell-counts">
                <span>持有 <b>{{ row.stock }}</b></span>
                <span>目标需 <b>{{ row.heartRequired }}</b></span>
                <span :class="{ 'is-lack': row.gap > 0 }">缺 <b>{{ row.gap }}</b></span>
                <span>本期 <em>+{{ row.acquired }}</em></span>
                <span>ETA <b>{{ row.etaDays == null ? '暂无' : formatEta(row.etaDays) }}</b></span>
              </div>
            </article>
          </div>
        </div>
      </template>
    </template>
    <div v-if="viewConfirmOpen" class="tracker-view-confirm" role="dialog" aria-modal="true" aria-label="保存确认">
      <div class="tracker-view-confirm-box">
        <h3>是否保存本次修改？</h3>
        <p>可以选择保存修改，或放弃并取消本次编辑。</p>
        <div class="tracker-view-confirm-actions">
          <button type="button" :disabled="viewSaving" @click="saveViewTarget(viewConfirmMode)">{{ viewSaving ? '保存中…' : '保存' }}</button>
          <button type="button" :disabled="viewSaving" @click="closeViewEditor">取消</button>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup>
import { computed, ref, watch, onBeforeUnmount } from 'vue'
import { Info, Pencil, RefreshCw, Star } from '@lucide/vue'
import { avatarUrl } from '../../api/request.js'
import { getAcquired, getCurrent } from '../../api/inventory.js'
import { deleteOperatorGrowthTarget, executeOperatorUpgrade, getOperatorGrowthTargets, previewOperatorUpgrade, putOperatorGrowthTarget } from '../../api/operator.js'
import { ITEM_CATALOG } from '../../data/inventory/catalog.js'
import {
  calculateLevelRequirements,
  calculateStarRequirements,
  calculateXiuweiRequirements,
  mergeRequirements,
  netRequirement,
  starLabelForStage,
  starStageFromLevel
} from '../../data/operatorRequirements.js'
import { matchesProfSubFilter, subProfList, subProfOptions } from '../../utils/operatorFilters.js'

const props = defineProps({
  accountId: { type: String, default: '' },
  game: { type: String, default: '' },
  currentEntries: { type: Array, default: () => [] },
  catalogEntries: { type: Array, default: () => [] },
  favoriteIds: { type: Object, default: () => new Set() },
  isLoggedIn: { type: Boolean, default: false },
  refreshKey: { type: Number, default: 0 },
  initialCurrentItems: { type: Object, default: () => ({}) },
  initialCurrentAgents: { type: Object, default: () => ({}) },
  currentInventoryReady: { type: Boolean, default: false }
})
const emit = defineEmits(['upgrade-applied'])

const loading = ref(false)
const trackerMode = ref('favorite')
const viewEditor = ref('')
const viewSaving = ref(false)
const viewDraft = ref({})
const viewSelectedOrder = ref([])
const viewSavedIds = ref(new Set())
const viewCompletedIds = ref(new Set())
const viewDirty = ref(false)
const viewConfirmOpen = ref(false)
const viewConfirmMode = ref('')
const viewSearch = ref('')
const viewProfFilter = ref('all')
const viewSubProfFilter = ref('all')
const viewOwnedOnly = ref(false)
const viewerGroups = ref([])
const VIEWER_UNGROUPED_ID = '__ungrouped__'
const viewerUngroupedIds = ref(new Set())
const viewerGroupName = ref('')
const viewerActiveGroupId = ref('')
const viewerEditingGroupId = ref('')
const viewerSelectedGroupIds = ref(new Set())
const viewerGroupPickerQuery = ref('')
const viewerGroupProfFilter = ref('all')
const viewerGroupSubProfFilter = ref('all')
const growthAllocationMode = ref('priority')
const growthPriorityIds = ref([])
const growthAllocationPanel = ref(false)
const growthDragIndex = ref(-1)
const error = ref('')
const currentItems = ref({})
const currentAgents = ref({})
const acquiredItems = ref({})
const acquiredAgents = ref({})
const rangeDays = 30
const rangeFrom = new Date(Date.now() - rangeDays * 86400000).toISOString()
const rangeTo = new Date().toISOString()
const targets = ref({})
const targetLoading = ref(false)
const targetError = ref('')
const targetNotice = ref('')
const targetBusyIds = ref(new Set())
const growthUpgradeKey = ref('')
const growthUpgradeField = ref('')
const growthUpgradePreviews = ref({})
const growthUpgradeErrors = ref({})
const growthUpgradeBusyKeys = ref(new Set())
const growthUpgradeExecuting = ref(false)
const growthUpgradeRequestSeqs = new Map()
const growthUpgradeExecutionKeys = new Map()
let targetLoadSeq = 0
let targetNoticeTimer = null
let inventoryLoadSeq = 0

const PROF_ICON_FILES = { 阳: 'yang.png', 阴: 'yin.png', 火: 'fire.png', 风: 'wind.png', 水: 'water.png', 地: 'earth.png', 混沌: 'chaos.png' }

const itemMap = computed(function () {
  const map = {}
  ITEM_CATALOG.forEach(function (item) { map[item.id] = item.name })
  props.catalogEntries.forEach(function (item) { if (item.id) map[item.id] = item.name || map[item.id] || item.id })
  return map
})

const currentMap = computed(function () {
  const map = {}
  props.currentEntries.forEach(function (entry) { map[entry.id] = entry })
  return map
})

const viewSearchOptions = computed(function () {
  const query = viewSearch.value.trim().toLowerCase()
  return props.catalogEntries.filter(function (entry) {
    if (!matchesProfSubFilter(entry, viewProfFilter.value, viewSubProfFilter.value)) return false
    if (viewOwnedOnly.value && !currentMap.value[entry.id]) return false
    if (query) {
      const hay = [entry.name, entry.alias, entry.id, entry.prof].filter(Boolean).concat(subProfList(entry)).join(' ').toLowerCase()
      if (hay.indexOf(query) === -1) return false
    }
    return true
  })
})

const viewProfOptions = ['阳', '阴', '火', '风', '水', '地', '混沌']
const viewSubProfOptions = computed(function () { return subProfOptions(props.catalogEntries) })
const selectedViewOperators = computed(function () {
  const has = {}
  Object.keys(viewDraft.value).forEach(function (id) { has[id] = true })
  const byId = {}
  props.catalogEntries.forEach(function (entry) { byId[entry.id] = entry })
  const order = viewSelectedOrder.value.filter(function (id) { return has[id] })
  order.forEach(function (id) { delete has[id] })
  Object.keys(has).forEach(function (id) { order.push(id) })
  const out = []
  order.forEach(function (id) { const entry = byId[id]; if (entry) out.push(entry) })
  return out
})
const hasViewFilters = computed(function () {
  return Boolean(viewSearch.value) || viewProfFilter.value !== 'all' || viewSubProfFilter.value !== 'all' || viewOwnedOnly.value
})
const starStages = [
  { value: 0, label: '未拥有' }
].concat(Array.from({ length: 24 }, function (_, index) {
  const value = index + 1
  return { value, label: starLabelForStage(starStageFromLevel(value)) }
}), [
  { value: 30, label: '五星' },
  { value: 31, label: '觉醒' }
])

const favoriteRows = computed(function () {
  const ids = props.favoriteIds instanceof Set ? props.favoriteIds : new Set()
  return props.catalogEntries.filter(function (entry) { return ids.has(entry.id) }).map(function (entry) {
    const current = currentMap.value[entry.id] || {}
    const target = targetFor(Object.assign({}, entry, current))
    const level = calculateLevelRequirements(current.level || 0, target.level, firstSubProf(entry))
    const xiuwei = calculateXiuweiRequirements(current.elite || 0, target.elite, xiuweiJob(entry.prof))
    const star = calculateStarRequirements(current.starLevel || 0, target.starLevel)
    const total = mergeRequirements(level, xiuwei, star)
    const stock = currentItems.value
    const net = netRequirement(total, stock)
    const ownedExperience = bookExperience(currentItems.value)
    const experienceGap = Math.max(level.experience - ownedExperience, 0)
    const heartOwned = Number(currentAgents.value[entry.id]) || 0
    const heartRequired = Math.max(Number(star.heart) || 0, Number(target.heartPaper) || 0)
    const heartGap = Math.max(heartRequired - heartOwned, 0)
    const gaps = net.gaps.slice()
    if (experienceGap) gaps.push({ id: '__experience__', required: level.experience, owned: ownedExperience, gap: experienceGap })
    const etaGaps = gaps.slice()
    if (heartGap) etaGaps.push({ id: '__heart__', agentId: entry.id, required: heartRequired, owned: heartOwned, gap: heartGap })
    const etaDays = etaForGaps(etaGaps)
    return Object.assign({}, entry, current, {
      owned: Boolean(current.level || current.elite || current.starLevel),
      level: Number(current.level) || 0,
      elite: Number(current.elite) || 0,
      starLevel: Number(current.starLevel) || 0,
      heartAcquired: Number(acquiredAgents.value[entry.id]) || 0,
      calculation: { level, xiuwei, star, total, net, experienceGap, heartOwned, heartRequired, heartGap, gaps, etaDays }
    })
  })
})

const growthMergedTargets = computed(function () {
  const merged = {}
  Object.keys(targets.value).forEach(function (id) {
    const saved = targets.value[id]
    if (saved && (saved.level != null || saved.elite != null)) merged[id] = saved
  })
  Object.keys(viewDraft.value).forEach(function (id) {
    const draft = viewDraft.value[id]
    if (!draft) return
    merged[id] = Object.assign({}, merged[id] || {}, {
      level: draft.level,
      elite: draft.elite,
      starLevel: draft.starLevel
    })
  })
  return merged
})

const heartMergedTargets = computed(function () {
  const merged = {}
  Object.keys(targets.value).forEach(function (id) {
    const saved = targets.value[id]
    if (saved && (saved.starLevel != null || saved.heartPaper != null)) merged[id] = saved
  })
  Object.keys(viewDraft.value).forEach(function (id) {
    const draft = viewDraft.value[id]
    if (!draft) return
    merged[id] = Object.assign({}, merged[id] || {}, {
      starLevel: draft.starLevel
    })
  })
  return merged
})

const growthRows = computed(function () {
  const ids = props.favoriteIds instanceof Set ? props.favoriteIds : new Set()
  const savedTargets = growthMergedTargets.value
  const rows = props.catalogEntries.filter(function (entry) {
    const target = savedTargets[entry.id]
    return !!target || viewDraft.value[entry.id]
  }).map(function (entry) { return buildGrowthRow(entry, savedTargets[entry.id]) })
  return allocateGrowthRows(rows).sort(function (a, b) {
    return (Number(ids.has(b.id)) - Number(ids.has(a.id))) || (Number(b.level) - Number(a.level)) || (Number(b.elite) - Number(a.elite)) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
})

const growthAllocationOrderRows = computed(function () {
  return growthAllocationOrder(growthRows.value)
})

const growthAggregateRows = computed(function () {
  const active = viewEditor.value === 'growth'
  const rows = props.catalogEntries.filter(function (entry) {
    if (active) return !!viewDraft.value[entry.id]
    return !!growthMergedTargets.value[entry.id]
  }).map(function (entry) { return buildGrowthRow(entry, growthMergedTargets.value[entry.id]) })
  return allocateGrowthRows(rows).sort(function (a, b) {
    return (Number(b.level) - Number(a.level)) || (Number(b.elite) - Number(a.elite)) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
})

const heartRows = computed(function () {
  const savedTargets = heartMergedTargets.value
  return props.catalogEntries.filter(function (entry) {
    const target = savedTargets[entry.id]
    return !!target || viewDraft.value[entry.id]
  }).map(function (entry) { return buildHeartRow(entry, savedTargets[entry.id]) }).sort(function (a, b) {
    return (Number(b.gap) - Number(a.gap)) || (Number(b.rarity) - Number(a.rarity)) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
})

const heartAggregateRows = computed(function () {
  const active = viewEditor.value === 'heart'
  return props.catalogEntries.filter(function (entry) {
    if (active) return !!viewDraft.value[entry.id]
    return !!heartMergedTargets.value[entry.id]
  }).map(function (entry) { return buildHeartRow(entry, heartMergedTargets.value[entry.id]) }).sort(function (a, b) {
    return (Number(b.gap) - Number(a.gap)) || (Number(b.rarity) - Number(a.rarity)) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
})

function buildGrowthRow(entry, targetOverride) {
  const current = currentMap.value[entry.id] || {}
  const target = targetFor(Object.assign({}, entry, current), targetOverride)
  const level = calculateLevelRequirements(current.level || 0, target.level, firstSubProf(entry))
  const xiuwei = calculateXiuweiRequirements(current.elite || 0, target.elite, xiuweiJob(entry.prof))
  const total = mergeRequirements(level, xiuwei)
  const net = netRequirement(total, currentItems.value)
  const ownedExperience = bookExperience(currentItems.value)
  const experienceGap = Math.max(level.experience - ownedExperience, 0)
  const gaps = net.gaps.slice()
  if (experienceGap) gaps.push({ id: '__experience__', required: level.experience, owned: ownedExperience, gap: experienceGap })
  const etaDays = etaForGaps(gaps)
  return Object.assign({}, entry, current, {
    owned: Boolean(current.level || current.elite || current.starLevel),
    level: Number(current.level) || 0,
    elite: Number(current.elite) || 0,
    starLevel: Number(current.starLevel) || 0,
    targetLevel: target.level,
    targetElite: target.elite,
    calculation: { level, xiuwei, total, net, experienceGap, gaps, etaDays }
  })
}

function allocateGrowthRows(rows) {
  if (!rows || !rows.length) return rows
  const allocatedMaps = rows.map(function () { return {} })
  const allocatedExperience = rows.map(function () { return 0 })
  const stock = currentItems.value

  if (growthAllocationMode.value === 'ratio') {
    const itemRequirements = rows.map(function (row) {
      const calculation = (row && row.calculation) || {}
      return (calculation.total && calculation.total.items) || {}
    })
    const requiredById = {}
    itemRequirements.forEach(function (items) {
      Object.keys(items).forEach(function (id) {
        requiredById[id] = (requiredById[id] || 0) + (Number(items[id]) || 0)
      })
    })
    const experienceRequirements = rows.map(function (row) {
      const calculation = (row && row.calculation) || {}
      return Number(calculation.level && calculation.level.experience) || 0
    })

    function allocate(requirements, available, totalRequired) {
      const result = requirements.map(function () { return 0 })
      if (totalRequired <= 0 || available <= 0) return result
      if (available >= totalRequired) {
        requirements.forEach(function (req, index) { result[index] = req })
        return result
      }
      requirements.forEach(function (req, index) {
        result[index] = Math.floor(req * available / totalRequired)
      })
      let remaining = available - result.reduce(function (sum, value) { return sum + value }, 0)
      if (remaining > 0) {
        const order = requirements.map(function (req, index) {
          return { index: index, remainder: req * available / totalRequired - result[index], req: req }
        }).sort(function (a, b) {
          return (b.remainder - a.remainder) || (b.req - a.req)
        })
        order.forEach(function (item) {
          if (remaining <= 0) return
          result[item.index] += 1
          remaining -= 1
        })
      }
      return result
    }

    Object.keys(requiredById).forEach(function (id) {
      const requirements = itemRequirements.map(function (items) { return Number(items[id]) || 0 })
      const allocated = allocate(requirements, Number(stock[id]) || 0, requiredById[id])
      allocated.forEach(function (owned, index) {
        if (owned > 0) allocatedMaps[index][id] = owned
      })
    })
    const totalExperience = experienceRequirements.reduce(function (sum, value) { return sum + value }, 0)
    const experienceAllocated = allocate(experienceRequirements, bookExperience(stock), totalExperience)
    experienceAllocated.forEach(function (owned, index) { allocatedExperience[index] = owned })
  } else {
    const ordered = growthAllocationOrder(rows)
    const remainingItems = Object.assign({}, stock)
    let remainingExperience = bookExperience(stock)
    ordered.forEach(function (row) {
      const index = rows.indexOf(row)
      const items = (row.calculation && row.calculation.total && row.calculation.total.items) || {}
      Object.keys(items).forEach(function (id) {
        const required = Number(items[id]) || 0
        const available = Number(remainingItems[id]) || 0
        const owned = Math.min(required, available)
        if (owned > 0) allocatedMaps[index][id] = owned
        remainingItems[id] = available - owned
      })
      const experienceRequired = Number(row.calculation && row.calculation.level && row.calculation.level.experience) || 0
      const ownedExperience = Math.min(experienceRequired, remainingExperience)
      allocatedExperience[index] = ownedExperience
      remainingExperience -= ownedExperience
    })
  }

  return rows.map(function (row, index) {
    const calculation = row.calculation || {}
    const allocatedItems = allocatedMaps[index] || {}
    const gaps = []
    Object.keys((calculation.total && calculation.total.items) || {}).forEach(function (id) {
      const required = Number(calculation.total.items[id]) || 0
      const owned = Number(allocatedItems[id]) || 0
      const gap = Math.max(required - owned, 0)
      if (gap > 0) gaps.push({ id: id, required: required, owned: owned, gap: gap })
    })
    const experienceRequired = Number(calculation.level && calculation.level.experience) || 0
    const experienceGap = Math.max(experienceRequired - allocatedExperience[index], 0)
    if (experienceGap > 0) {
      gaps.push({ id: '__experience__', required: experienceRequired, owned: allocatedExperience[index], gap: experienceGap })
    }
    return Object.assign({}, row, {
      allocatedItems: allocatedItems,
      allocatedExperience: allocatedExperience[index],
      allocationReady: !gaps.length,
      calculation: Object.assign({}, calculation, { gaps: gaps, experienceGap: experienceGap })
    })
  })
}

function buildHeartRow(entry, targetOverride) {
  const current = currentMap.value[entry.id] || {}
  const target = targetFor(Object.assign({}, entry, current), targetOverride)
  const star = calculateStarRequirements(current.starLevel || 0, target.starLevel)
  const stock = Number(currentAgents.value[entry.id]) || 0
  const heartRequired = Math.max(star.heart || 0, Number(target.heartPaper) || 0)
  const gap = Math.max(heartRequired - stock, 0)
  const acquired = Number(acquiredAgents.value[entry.id]) || 0
  const rate = acquired / rangeDays
  return Object.assign({}, entry, current, {
    owned: Boolean(current.level || current.elite || current.starLevel),
    level: Number(current.level) || 0,
    starLevel: Number(current.starLevel) || 0,
    currentStar: Number(current.starLevel) || 0,
    targetStar: target.starLevel,
    targetStarLabel: starLabel(target.starLevel),
    targetHeartPaper: target.heartPaper,
    stock: stock,
    heartRequired: heartRequired,
    gap: gap,
    acquired: acquired,
    heartRate: rate,
    etaDays: gap > 0 ? (rate > 0 ? gap / rate : null) : 0
  })
}

const growthAggregate = computed(function () {
  const requirements = []
  const gapsById = {}
  let experienceGap = 0
  growthAggregateRows.value.forEach(function (row) {
    requirements.push(row.calculation.level, row.calculation.xiuwei)
    experienceGap += Number(row.calculation.experienceGap) || 0
    ;(row.calculation.gaps || []).forEach(function (gap) {
      const key = gap.id
      const existing = gapsById[key]
      if (existing) {
        existing.required += Number(gap.required) || 0
        existing.owned += Number(gap.owned) || 0
        existing.gap += Number(gap.gap) || 0
      } else {
        gapsById[key] = { id: key, required: Number(gap.required) || 0, owned: Number(gap.owned) || 0, gap: Number(gap.gap) || 0 }
      }
    })
  })
  const total = mergeRequirements.apply(null, requirements)
  const gaps = Object.keys(gapsById).map(function (key) { return gapsById[key] })
  return { total, gaps, experienceGap, etaDays: etaForGaps(gaps), gapCount: gaps.length }
})

const heartAggregate = computed(function () {
  let required = 0
  let stock = 0
  let acquired = 0
  let gap = 0
  heartAggregateRows.value.forEach(function (row) {
    required += row.heartRequired
    stock += row.stock
    acquired += row.acquired
    gap += row.gap
  })
  const rate = acquired / rangeDays
  return { required, stock, acquired, gap, etaDays: gap > 0 ? (rate > 0 ? gap / rate : null) : 0 }
})

const growthAggregateEtaLabel = computed(function () {
  if (!growthAggregate.value.gapCount) return '当前练度目标无需等待'
  if (growthAggregate.value.etaDays == null) return '暂无整体 ETA：至少一项缺口没有对应流水'
  return '按当前速度，最慢材料约 ' + formatEta(growthAggregate.value.etaDays)
})

const heartAggregateEtaLabel = computed(function () {
  if (heartAggregate.value.gap <= 0) return '当前心纸目标已备齐'
  if (heartAggregate.value.etaDays == null) return '暂无 ETA：没有足够的心纸流水'
  return '按当前速度，最慢密探约 ' + formatEta(heartAggregate.value.etaDays)
})

const groupById = computed(function () {
  const map = {}
  viewerGroups.value.forEach(function (group) { if (group && group.id) map[group.id] = group })
  return map
})

const viewerGroupPickerOptions = computed(function () {
  const query = viewerGroupPickerQuery.value.trim().toLowerCase()
  return props.catalogEntries.filter(function (entry) {
    if (!matchesProfSubFilter(entry, viewerGroupProfFilter.value, viewerGroupSubProfFilter.value)) return false
    if (query) {
      const hay = [entry.name, entry.alias, entry.id, entry.prof].filter(Boolean).concat(subProfList(entry)).join(' ').toLowerCase()
      if (hay.indexOf(query) === -1) return false
    }
    return true
  }).sort(function (a, b) {
    return Number(b.rarity) - Number(a.rarity) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
})

const viewerGroupSubProfOptions = computed(function () { return subProfOptions(props.catalogEntries) })
const viewerActiveGroup = computed(function () {
  if (viewerActiveGroupId.value === VIEWER_UNGROUPED_ID) return { id: VIEWER_UNGROUPED_ID, name: '未分组', operatorIds: [] }
  return groupById.value[viewerActiveGroupId.value] || null
})

const groupMemberEntries = computed(function () {
  const byId = {}
  props.catalogEntries.forEach(function (entry) { byId[entry.id] = entry })
  const map = {}
  viewerGroups.value.forEach(function (group) {
    map[group.id] = (group.operatorIds || []).map(function (id) { return byId[id] }).filter(Boolean)
  })
  map[VIEWER_UNGROUPED_ID] = Array.from(viewerUngroupedIds.value).map(function (id) { return byId[id] }).filter(Boolean)
  return map
})

const viewerActiveGroupEntries = computed(function () {
  return groupMemberEntries.value[viewerActiveGroupId.value] || []
})

const viewerGroupStatCards = computed(function () {
  return viewerGroups.value.map(function (group) {
    const rows = groupRowsFor(group, null).map(displayGroupRow)
    return {
      id: group.id,
      name: group.name || group.id,
      group: group,
      count: (group.operatorIds || []).length,
      rows: rows,
      aggregate: aggregateForRows(groupRowsFor(group, null))
    }
  })
})

const selectedGroupIds = computed(function () {
  return viewerGroups.value.map(function (group) { return group.id }).filter(function (id) { return viewerSelectedGroupIds.value.has(id) })
})

const selectedGroupAggregate = computed(function () {
  const ids = selectedGroupIds.value
  if (!ids.length) return null
  const rows = mergeRowsFor(ids)
  return aggregateForRows(rows)
})

const selectedGroupRows = computed(function () {
  const ids = selectedGroupIds.value
  if (!ids.length) return []
  return mergeRowsFor(ids).map(displayGroupRow)
})

const allGroupBoard = computed(function () {
  const allIds = viewerGroups.value.map(function (group) { return group.id })
  const rows = mergeRowsFor(allIds)
  const groupSet = new Set()
  viewerGroups.value.forEach(function (group) {
    ;(group.operatorIds || []).forEach(function (id) { groupSet.add(id) })
  })
  props.catalogEntries.forEach(function (entry) {
    if (groupSet.has(entry.id)) return
    if (!targets.value[entry.id]) return
    rows.push(buildGrowthRow(entry, targets.value[entry.id]))
  })
  const ordered = rows.slice().sort(function (a, b) {
    return (Number(b.level) - Number(a.level)) || (Number(b.elite) - Number(a.elite)) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
  return {
    rows: ordered.map(displayGroupRow),
    aggregate: aggregateForRows(ordered)
  }
})

function groupRowsFor(group, maxTarget) {
  const entries = groupMemberEntries.value[group.id] || []
  if (!maxTarget) maxTarget = mergeTargetsForIds(group.operatorIds || [])
  return entries.map(function (entry) {
    const m = maxTarget[entry.id] || {}
    return buildGrowthRow(entry, m)
  }).sort(function (a, b) {
    return (Number(b.level) - Number(a.level)) || (Number(b.elite) - Number(a.elite)) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
}

function mergeTargetsForIds(ids) {
  const merged = {}
  ;(ids || []).forEach(function (id) {
    if (!targets.value[id]) return
    const current = merged[id]
    const incoming = targets.value[id]
    merged[id] = {
      level: Math.max(Number(current && current.level) || 0, Number(incoming.level) || 0),
      elite: Math.max(Number(current && current.elite) || 0, Number(incoming.elite) || 0),
      starLevel: Math.max(Number(current && current.starLevel) || 0, Number(incoming.starLevel) || 0),
      heartPaper: Math.max(Number(current && current.heartPaper) || 0, Number(incoming.heartPaper) || 0)
    }
  })
  return merged
}

function mergeRowsFor(ids) {
  const merged = mergeTargetsForIds(ids)
  const byId = {}
  props.catalogEntries.forEach(function (entry) { byId[entry.id] = entry })
  return Object.keys(merged).map(function (id) {
    const entry = byId[id]
    if (!entry) return null
    return buildGrowthRow(entry, merged[id])
  }).filter(Boolean)
}

function aggregateForRows(rows) {
  const requirements = []
  let experience = 0
  rows.forEach(function (row) {
    requirements.push(row.calculation.level, row.calculation.xiuwei)
    experience += Number(row.calculation.level.experience) || 0
  })
  const total = mergeRequirements.apply(null, requirements)
  const net = netRequirement(total, currentItems.value)
  const gaps = net.gaps.slice()
  const owned = bookExperience(currentItems.value)
  const experienceGap = Math.max(experience - owned, 0)
  if (experienceGap) gaps.push({ id: '__experience__', required: experience, owned: owned, gap: experienceGap })
  return {
    rows: rows,
    total: total,
    gaps: gaps,
    experienceGap: experienceGap,
    etaDays: etaForGaps(gaps),
    gapCount: gaps.length
  }
}

function displayGroupRow(row) {
  return Object.assign({}, row, {
    targetText: 'Lv' + row.targetLevel + ' + ' + row.targetElite + (row.targetStar > row.starLevel ? ' · ' + starLabel(row.targetStar) : ''),
    gapText: (row.calculation.gaps || []).map(function (gap) { return itemName(gap.id) + '×' + formatNumber(gap.gap) }).join('、') || '已备齐'
  })
}

const aggregatePlan = computed(function () {
  const requirements = []
  let experience = 0
  const heartGaps = []
  favoriteRows.value.forEach(function (row) {
    requirements.push(row.calculation.level, row.calculation.xiuwei, row.calculation.star)
    experience += Number(row.calculation.level.experience) || 0
    if (row.calculation.heartGap) {
      heartGaps.push({
        id: '__heart__',
        agentId: row.id,
        required: row.calculation.heartRequired,
        owned: row.calculation.heartOwned,
        gap: row.calculation.heartGap
      })
    }
  })
  const total = mergeRequirements(...requirements)
  const net = netRequirement(total, currentItems.value)
  const materialGaps = net.gaps.slice()
  const experienceGap = Math.max(experience - bookExperience(currentItems.value), 0)
  if (experienceGap) materialGaps.push({ id: '__experience__', required: experience, owned: bookExperience(currentItems.value), gap: experienceGap })
  const heartGap = heartGaps.reduce(function (sum, gap) { return sum + gap.gap }, 0)
  return {
    total,
    materialGaps,
    heartGaps,
    heartGap,
    etaDays: etaForGaps(materialGaps.concat(heartGaps))
  }
})

const ownedFavoriteCount = computed(function () { return favoriteRows.value.filter(function (row) { return row.owned }).length })
const totalHeartStock = computed(function () { return favoriteRows.value.reduce(function (sum, row) { return sum + (Number(currentAgents.value[row.id]) || 0) }, 0) })
const totalHeartAcquired = computed(function () { return favoriteRows.value.reduce(function (sum, row) { return sum + (Number(acquiredAgents.value[row.id]) || 0) }, 0) })
const totalGapCount = computed(function () {
  return aggregatePlan.value.materialGaps.length + aggregatePlan.value.heartGaps.length
})
const aggregateEtaLabel = computed(function () {
  if (!aggregatePlan.value.materialGaps.length && !aggregatePlan.value.heartGaps.length) return '当前目标无需等待'
  if (aggregatePlan.value.etaDays == null) return '暂无整体 ETA：至少一项缺口没有对应流水'
  return '按当前速度，最慢材料约 ' + formatEta(aggregatePlan.value.etaDays)
})

function targetStorageKey() { return 'yuanhub:operator-targets:' + props.accountId }

function targetMigrationKey() { return 'yuanhub:operator-targets-migrated:v1:' + props.accountId }

function viewerGroupsStorageKey() { return 'yuanhub:growth-groups:' + props.accountId }

function allocationPriorityStorageKey() { return 'yuanhub:growth-allocation-priority:' + props.accountId }

function readGrowthAllocationPriority() {
  if (typeof localStorage === 'undefined' || !props.accountId) return []
  try {
    const parsed = JSON.parse(localStorage.getItem(allocationPriorityStorageKey()) || '[]')
    return Array.isArray(parsed) ? parsed.filter(function (id) { return typeof id === 'string' }) : []
  } catch (_) {
    return []
  }
}

function cacheGrowthAllocationPriority() {
  if (typeof localStorage === 'undefined' || !props.accountId) return
  try { localStorage.setItem(allocationPriorityStorageKey(), JSON.stringify(growthPriorityIds.value)) } catch (_) {}
}

function growthMaterialAmount(row) {
  const calculation = (row && row.calculation) || {}
  const total = calculation.total || {}
  const items = total.items || {}
  return Object.keys(items).reduce(function (sum, id) { return sum + (Number(items[id]) || 0) }, 0) +
    (Number(total.money) || 0) / 100000 +
    (Number(calculation.level && calculation.level.experience) || 0) / 100
}

function growthRarityLabel(rarity) {
  const map = { 5: '绝密', 4: '机密', 3: '隐密' }
  return map[Number(rarity)] || ('品质 ' + rarity)
}

function growthAllocationOrder(rows) {
  const byId = {}
  rows.forEach(function (row) { byId[row.id] = row })
  const manual = growthPriorityIds.value.filter(function (id) { return byId[id] })
  const rest = rows.filter(function (row) { return manual.indexOf(row.id) === -1 }).slice().sort(function (a, b) {
    return (Number(b.rarity) - Number(a.rarity)) || (growthMaterialAmount(b) - growthMaterialAmount(a)) || String(a.name || '').localeCompare(String(b.name || ''), 'zh-CN')
  })
  return manual.map(function (id) { return byId[id] }).concat(rest)
}

function setGrowthAllocationMode(mode) {
  growthAllocationMode.value = mode === 'ratio' ? 'ratio' : 'priority'
}

function resetGrowthPriority() {
  growthPriorityIds.value = []
  cacheGrowthAllocationPriority()
}

function reorderGrowthPriority(from, to) {
  const current = growthAllocationOrderRows.value.map(function (row) { return row.id })
  if (from < 0 || from >= current.length || to < 0 || to >= current.length || from === to) return
  const next = current.slice()
  next.splice(to, 0, next.splice(from, 1)[0])
  growthPriorityIds.value = next
  cacheGrowthAllocationPriority()
}

function moveGrowthPriority(id, direction) {
  const current = growthAllocationOrderRows.value.map(function (row) { return row.id })
  const from = current.indexOf(id)
  reorderGrowthPriority(from, from + direction)
}

function startGrowthDrag(index) {
  growthDragIndex.value = index
}

function endGrowthDrag() {
  growthDragIndex.value = -1
}

function dropGrowthPriority(index) {
  const from = growthDragIndex.value
  growthDragIndex.value = -1
  if (from < 0 || from === index) return
  reorderGrowthPriority(from, index)
}

function loadViewerGroups() {
  if (typeof localStorage === 'undefined' || !props.accountId) return
  try {
    const parsed = JSON.parse(localStorage.getItem(viewerGroupsStorageKey()) || '{}')
    if (Array.isArray(parsed)) {
      viewerGroups.value = parsed
      viewerUngroupedIds.value = new Set()
    } else {
      if (Array.isArray(parsed && parsed.groups)) viewerGroups.value = parsed.groups
      const raw = Array.isArray(parsed && parsed.ungrouped) ? parsed.ungrouped : []
      viewerUngroupedIds.value = new Set(raw)
    }
  } catch (_) {}
}

function persistViewerGroups() {
  if (typeof localStorage === 'undefined' || !props.accountId) return
  try {
    const data = { groups: viewerGroups.value, ungrouped: Array.from(viewerUngroupedIds.value) }
    localStorage.setItem(viewerGroupsStorageKey(), JSON.stringify(data))
  } catch (_) {}
}

watch([viewerGroups, viewerUngroupedIds], persistViewerGroups, { deep: true })
watch(function () { return props.accountId }, function () {
  viewerGroups.value = []
  viewerUngroupedIds.value = new Set()
  growthPriorityIds.value = readGrowthAllocationPriority()
  loadViewerGroups()
})

function createViewerGroup() {
  const name = String(viewerGroupName.value || '').trim() || ('分组 ' + (viewerGroups.value.length + 1))
  const group = { id: 'g' + Date.now().toString(36) + Math.random().toString(36).slice(2, 7), name: name, operatorIds: [] }
  viewerGroups.value = viewerGroups.value.concat(group)
  viewerGroupName.value = ''
  viewerActiveGroupId.value = group.id
  viewerSelectedGroupIds.value = new Set(viewerSelectedGroupIds.value).add(group.id)
}

function updateViewerGroupName(group) {
  const next = viewerGroups.value.map(function (item) {
    if (item !== group) return item
    return Object.assign({}, item, { name: String(group.name || '').trim() || item.name })
  })
  viewerGroups.value = next.slice()
  viewerGroupName.value = ''
}

function toggleViewerGroupMember(group, id) {
  const member = viewerGroupHasMember(group, id)
  if (group && group.id === VIEWER_UNGROUPED_ID) {
    if (member) {
      const next = new Set(viewerUngroupedIds.value)
      next.delete(id)
      viewerUngroupedIds.value = next
    } else {
      viewerUngroupedIds.value = new Set(viewerUngroupedIds.value).add(id)
      viewerGroups.value.forEach(function (item) { removeGroupMember(item.id, id) })
    }
    return
  }
  if (member) removeGroupMember(group.id, id)
  else addGroupMember(group.id, id)
}

function addToEditorAndGroup(entry) {
  if (!entry || !entry.id) return
  if (viewDraft.value[entry.id]) {
    removeViewOperator(entry.id)
    viewerUngroupedIds.value = new Set(viewerUngroupedIds.value)
    viewerUngroupedIds.value.delete(entry.id)
    viewerGroups.value.forEach(function (group) {
      if ((group.operatorIds || []).indexOf(entry.id) !== -1) removeGroupMember(group.id, entry.id)
    })
    return
  }
  selectViewOperator(entry)
  if (viewerActiveGroup.value) {
    if (viewerActiveGroup.value.id === VIEWER_UNGROUPED_ID) {
      viewerUngroupedIds.value = new Set(viewerUngroupedIds.value).add(entry.id)
      viewerGroups.value.forEach(function (group) { removeGroupMember(group.id, entry.id) })
    } else {
      addGroupMember(viewerActiveGroup.value.id, entry.id)
    }
  }
}

function addGroupMember(groupId, id) {
  const ungrouped = new Set(viewerUngroupedIds.value)
  ungrouped.delete(id)
  viewerUngroupedIds.value = ungrouped
  const next = viewerGroups.value.map(function (group) {
    if (group.id !== groupId) return group
    const ids = (group.operatorIds || []).slice()
    if (ids.indexOf(id) === -1) ids.push(id)
    return Object.assign({}, group, { operatorIds: ids })
  })
  viewerGroups.value = next.slice()
}

function removeGroupMember(groupId, id) {
  const next = viewerGroups.value.map(function (group) {
    if (group.id !== groupId) return group
    const ids = (group.operatorIds || []).filter(function (value) { return value !== id })
    return Object.assign({}, group, { operatorIds: ids })
  })
  viewerGroups.value = next.slice()
}

async function removeViewerGroup(id) {
  const group = viewerGroups.value.find(function (item) { return item.id === id })
  const memberIds = (group && group.operatorIds) ? group.operatorIds.slice() : []
  viewerGroups.value = viewerGroups.value.filter(function (item) { return item.id !== id })
  if (viewerActiveGroupId.value === id) viewerActiveGroupId.value = VIEWER_UNGROUPED_ID
  const selected = new Set(viewerSelectedGroupIds.value)
  selected.delete(id)
  viewerSelectedGroupIds.value = selected
  for (const memberId of memberIds) {
    const stillInGroup = viewerGroups.value.some(function (item) { return (item.operatorIds || []).indexOf(memberId) !== -1 })
    const ungrouped = new Set(viewerUngroupedIds.value)
    ungrouped.delete(memberId)
    viewerUngroupedIds.value = ungrouped
    removeViewOperator(memberId)
    const saved = targets.value[memberId]
    if (stillInGroup || !saved) continue
    viewSavedIds.value = new Set(viewSavedIds.value).add(memberId)
    try {
      await deleteOperatorGrowthTarget({ accountId: props.accountId, operatorId: memberId, expectedRevision: Number(saved.revision) || 0 })
      targets.value = Object.assign({}, targets.value)
      delete targets.value[memberId]
    } catch (err) {
      if (err && err.code === 'growth_target_revision_conflict') await loadTargets()
      targetError.value = targetErrorMessage(err, '养成目标删除失败')
    } finally {
      const next = new Set(viewSavedIds.value)
      next.delete(memberId)
      viewSavedIds.value = next
    }
  }
  cacheTargets()
  targetNotice.value = '分组及组内目标已删除'
  if (targetNoticeTimer != null) clearTimeout(targetNoticeTimer)
  targetNoticeTimer = setTimeout(function () { targetNotice.value = '' }, 1800)
}

function clearActiveGroupView() {
  const active = viewerActiveGroup.value
  if (!active) return
  if (active.id === VIEWER_UNGROUPED_ID) {
    const ids = Array.from(viewerUngroupedIds.value)
    ids.forEach(function (id) { removeViewOperator(id) })
    viewerUngroupedIds.value = new Set()
    return
  }
  const ids = (active.operatorIds || []).slice()
  ids.forEach(function (id) {
    const next = new Set(viewerUngroupedIds.value)
    next.delete(id)
    viewerUngroupedIds.value = next
    removeViewOperator(id)
  })
  const next = viewerGroups.value.map(function (item) {
    if (item.id !== active.id) return item
    return Object.assign({}, item, { operatorIds: [] })
  })
  viewerGroups.value = next.slice()
}

function toggleViewerGroupSelected(id) {
  const next = new Set(viewerSelectedGroupIds.value)
  if (next.has(id)) next.delete(id)
  else next.add(id)
  viewerSelectedGroupIds.value = next
}

function viewerGroupHasMember(group, id) {
  if (group && group.id === VIEWER_UNGROUPED_ID) return viewerUngroupedIds.value.has(id)
  return !!(group && (group.operatorIds || []).indexOf(id) !== -1)
}

function memberName(id) {
  const entry = props.catalogEntries.find(function (item) { return item.id === id })
  return entry ? (entry.name || id) : id
}

function groupTargetText(groupId, id) {
  const group = groupById.value[groupId]
  const target = group
    ? mergeTargetsForIds([id])[id]
    : (viewerUngroupedIds.value.has(id) ? targets.value[id] : null)
  const entry = props.catalogEntries.find(function (item) { return item.id === id })
  if (!target || !entry) return ''
  const current = currentMap.value[id] || {}
  const row = buildGrowthRow(Object.assign({}, entry, current), target)
  return displayGroupRow(row).targetText
}

const selectedGroupIdsLabel = computed(function () {
  const names = selectedGroupIds.value.map(function (id) { return (groupById.value[id] || {}).name || id })
  return names.join(' + ')
})

function readLocalTargets() {
  if (typeof localStorage === 'undefined' || !props.accountId) return {}
  try {
    const parsed = JSON.parse(localStorage.getItem(targetStorageKey()) || '{}')
    return parsed && typeof parsed === 'object' ? parsed : {}
  } catch (_) { return {} }
}

function cacheTargets() {
  if (typeof localStorage === 'undefined' || !props.accountId) return
  try { localStorage.setItem(targetStorageKey(), JSON.stringify(targets.value)) } catch (_) {}
}

function targetErrorMessage(err, fallback) {
  if (err && err.code === 'growth_target_revision_conflict') return '养成目标已在其他页面更新，已重新同步'
  if (err && err.code === 'invalid_growth_target') return '目标数值或组合不符合要求'
  return err && err.message ? err.message : fallback
}

function normalizedTargetItem(item) {
  return {
    level: item.level == null ? null : Number(item.level),
    elite: item.elite == null ? null : Number(item.elite),
    starLevel: (item.star_level != null ? item.star_level : item.starLevel) == null ? null : Number(item.star_level != null ? item.star_level : item.starLevel),
    heartPaper: (item.heart_paper != null ? item.heart_paper : item.heartPaper) == null ? null : Number(item.heart_paper != null ? item.heart_paper : item.heartPaper),
    revision: Number(item.revision) || 0
  }
}

function applyTargetItem(id, item) {
  const operatorId = id || (item && (item.operator_id || item.operatorId))
  if (!operatorId || !item) return
  targets.value = Object.assign({}, targets.value, { [operatorId]: normalizedTargetItem(item) })
}

async function migrateLocalTargets(targetAccount, remoteIds) {
  if (typeof localStorage === 'undefined' || localStorage.getItem(targetMigrationKey()) === 'done') return
  const local = readLocalTargets()
  const ids = Object.keys(local).filter(function (id) { return !remoteIds.has(id) })
  for (const id of ids) {
    const saved = local[id] || {}
    const body = { expected_revision: 0 }
    if (saved.level != null) body.level = Number(saved.level)
    if (saved.elite != null) body.elite = Number(saved.elite)
    if (saved.starLevel != null || saved.star_level != null) body.star_level = Number(saved.starLevel != null ? saved.starLevel : saved.star_level)
    if (saved.heartPaper != null || saved.heart_paper != null) body.heart_paper = Number(saved.heartPaper != null ? saved.heartPaper : saved.heart_paper)
    if (Object.keys(body).length === 1) continue
    const item = await putOperatorGrowthTarget({ accountId: targetAccount, operatorId: id, target: body })
    if (props.accountId !== targetAccount) return
    applyTargetItem(id, item)
  }
  localStorage.setItem(targetMigrationKey(), 'done')
}

async function loadTargets() {
  targets.value = {}
  targetError.value = ''
  if (!props.isLoggedIn || !props.accountId) {
    targetLoadSeq += 1
    targetLoading.value = false
    return
  }
  const targetAccount = props.accountId
  const seq = ++targetLoadSeq
  targetLoading.value = true
  try {
    const data = await getOperatorGrowthTargets(targetAccount)
    if (seq !== targetLoadSeq || props.accountId !== targetAccount) return
    const items = Array.isArray(data && data.items) ? data.items : []
    const mapped = {}
    const remoteIds = new Set()
    items.forEach(function (item) {
      const id = item && (item.operator_id || item.operatorId)
      if (!id) return
      remoteIds.add(id)
      mapped[id] = normalizedTargetItem(item)
    })
    targets.value = mapped
    await migrateLocalTargets(targetAccount, remoteIds)
    if (seq !== targetLoadSeq || props.accountId !== targetAccount) return
    cacheTargets()
  } catch (err) {
    if (seq !== targetLoadSeq || props.accountId !== targetAccount) return
    targets.value = readLocalTargets()
    targetError.value = targetErrorMessage(err, '养成目标同步失败，当前显示本地缓存')
  } finally {
    if (seq === targetLoadSeq && props.accountId === targetAccount) targetLoading.value = false
  }
}

function defaultTarget(row) {
  return { level: 100, elite: 17, starLevel: 7, heartPaper: null, revision: 0 }
}

function maxEliteForLevel(level) {
  const n = Math.min(100, Math.max(0, Math.trunc(Number(level) || 0)))
  if (n <= 0) return 0
  if (n >= 40) return Math.min(17, Math.max(0, Math.floor(n / 5) - 3))
  if (n >= 30) return 4
  if (n >= 15) return 3
  if (n >= 10) return 2
  return 1
}

function targetFor(row, override) {
  if (!row || !row.id) return defaultTarget(row)
  const saved = override || targets.value[row.id] || defaultTarget(row)
  const defaults = defaultTarget(row)
  const currentLevel = Number(row.level) || 0
  const currentElite = Number(row.elite) || 0
  const level = Math.max(currentLevel, Math.min(100, Number(saved.level == null ? defaults.level : saved.level) || 0))
  const eliteLimit = maxEliteForLevel(level)
  const elite = Math.max(currentElite, Math.min(eliteLimit, Number(saved.elite == null ? defaults.elite : saved.elite) || 0))
  const savedStarLevel = Math.min(31, Math.max(0, Number(saved.starLevel == null ? defaults.starLevel : saved.starLevel) || 0))
  const currentStarLevel = Number(row.starLevel) || 0
  const starLevel = starStage(savedStarLevel) < starStage(currentStarLevel) ? currentStarLevel : savedStarLevel
  const heartPaper = saved.heartPaper == null ? null : Math.max(0, Math.min(1000000, Number(saved.heartPaper) || 0))
  return { level, elite, starLevel, heartPaper, revision: Number(saved.revision) || 0 }
}

async function setTarget(row, field, event) {
  const id = row && row.id
  if (!id || targetBusyIds.value.has(id)) return
  const current = currentMap.value[id] || {}
  const target = Object.assign({}, targetFor(row))
  const raw = event && event.target ? event.target.value : ''
  const max = field === 'level' ? 100 : field === 'elite' ? 17 : field === 'heartPaper' ? 1000000 : 31
  const currentValue = field === 'heartPaper' ? 0 : Number(current[field]) || 0
  target[field] = Math.min(max, Math.max(currentValue, Number(raw) || 0))
  const eliteLimit = maxEliteForLevel(target.level || 0)
  if (field === 'elite') target[field] = Math.max(Number(current.elite) || 0, Math.min(target[field], eliteLimit))
  if (field === 'level') target.elite = Math.max(Number(current.elite) || 0, Math.min(target.elite, eliteLimit))
  const previous = targets.value[id]
  targets.value = Object.assign({}, targets.value, { [id]: target })
  targetBusyIds.value = new Set(targetBusyIds.value).add(id)
  targetError.value = ''
  targetNotice.value = ''
  try {
    const body = { expected_revision: Number(previous && previous.revision) || 0 }
    body[field === 'starLevel' ? 'star_level' : field === 'heartPaper' ? 'heart_paper' : field] = target[field]
    if (field === 'level' && target.elite !== (previous && previous.elite)) body.elite = target.elite
    const item = await putOperatorGrowthTarget({ accountId: props.accountId, operatorId: id, target: body })
    applyTargetItem(id, item)
    cacheTargets()
    targetNotice.value = (row.name || id) + '的养成目标已同步'
    if (targetNoticeTimer != null) clearTimeout(targetNoticeTimer)
    targetNoticeTimer = setTimeout(function () { targetNotice.value = '' }, 1800)
  } catch (err) {
    if (err && err.code === 'growth_target_revision_conflict') await loadTargets()
    else targets.value = Object.assign({}, targets.value, { [id]: previous || defaultTarget(row) })
    targetError.value = targetErrorMessage(err, '养成目标保存失败')
  } finally {
    const next = new Set(targetBusyIds.value)
    next.delete(id)
    targetBusyIds.value = next
  }
}

function openViewEditor(mode, operatorId) {
  viewEditor.value = mode
  viewDirty.value = false
  viewConfirmOpen.value = false
  viewConfirmMode.value = ''
  viewSearch.value = ''
  viewProfFilter.value = 'all'
  viewSubProfFilter.value = 'all'
  viewOwnedOnly.value = false
  targetError.value = ''
  targetNotice.value = ''
  viewDraft.value = {}
  viewSelectedOrder.value = []
  viewSavedIds.value = new Set()
  viewCompletedIds.value = new Set()
  preloadViewSaved(mode)
  if (operatorId) {
    const entry = props.catalogEntries.find(function (item) { return item.id === operatorId })
    if (entry) selectViewOperator(entry)
  }
}

function toggleViewEditor(mode) {
  if (viewEditor.value === mode) {
    if (!viewDirty.value) {
      closeViewEditor()
      return
    }
    viewConfirmMode.value = mode
    viewConfirmOpen.value = true
    return
  }
  openViewEditor(mode)
}

function preloadViewSaved(mode) {
  const savedIds = props.catalogEntries.filter(function (entry) {
    const target = targets.value[entry.id]
    if (!target) return false
    return mode === 'growth'
      ? (target.level != null || target.elite != null)
      : (target.starLevel != null || target.heartPaper != null)
  })
  savedIds.reverse().forEach(function (entry) {
    if (viewDraft.value[entry.id]) return
    viewDraft.value = Object.assign({}, viewDraft.value, { [entry.id]: viewDraftFor(entry) })
    viewSelectedOrder.value = [entry.id].concat(viewSelectedOrder.value.filter(function (item) { return item !== entry.id }))
  })
}

function viewDraftFor(entry) {
  const id = entry && entry.id
  if (!id) return null
  if (viewDraft.value[id]) return viewDraft.value[id]
  const current = currentMap.value[id] || {}
  const saved = targets.value[id] || {}
  const draft = {
    level: saved.level != null ? Number(saved.level) : 100,
    elite: saved.elite != null ? Number(saved.elite) : 17,
    starLevel: saved.starLevel != null ? Number(saved.starLevel) : Number(current.starLevel) || 7
  }
  viewDraft.value = Object.assign({}, viewDraft.value, { [id]: draft })
  return viewDraft.value[id]
}

const opGapOpenIds = ref(new Set())

function currentFor(op) {
  const current = currentMap.value[op.id] || {}
  return { level: Number(current.level) || 0, elite: Number(current.elite) || 0, starLevel: Number(current.starLevel) || 0 }
}

function entryOwned(entry) {
  const current = currentMap.value[entry && entry.id] || {}
  return Boolean(Number(current.level) || Number(current.elite) || Number(current.starLevel))
}

function toggleOpGap(id) {
  const next = new Set(opGapOpenIds.value)
  if (next.has(id)) next.delete(id)
  else next.add(id)
  opGapOpenIds.value = next
}

function growthUpgradeFields(row) {
  if (!row || !row.id) return []
  const fields = []
  const level = Number(row.level) || 0
  const elite = Number(row.elite) || 0
  if (level < (Number(row.targetLevel) || 0)) fields.push('level')
  if (elite < (Number(row.targetElite) || 0) && elite < maxEliteForLevel(level)) fields.push('elite')
  return fields
}

function growthHasUpgrade(row, field) {
  return growthUpgradeFields(row).indexOf(field) !== -1
}

function growthStepTarget(row, field) {
  const level = Number(row.level) || 0
  const elite = Number(row.elite) || 0
  if (field === 'level') return Math.min(Number(row.targetLevel) || 0, level + 5)
  const cap = Math.min(Number(row.targetElite) || 0, maxEliteForLevel(level))
  return Math.min(cap, elite + 1)
}

function growthUpgradeStepDelta(row, field) {
  return Math.max(0, growthStepTarget(row, field) - (field === 'level' ? Number(row.level) || 0 : Number(row.elite) || 0))
}

function growthFieldLabel(field) {
  return field === 'level' ? '等级' : '修为'
}

function trackerGame(row) {
  if (props.game) return props.game
  const games = row && row.games
  if (Array.isArray(games) && games.length) return games[games.length - 1]
  return props.game
}

function growthUpgradeReady(row, field) {
  if (!row || !row.id || !props.isLoggedIn || !props.accountId || !trackerGame(row)) return false
  if (!growthHasUpgrade(row, field)) return false
  const target = growthStepTarget(row, field)
  const itemsStock = row.allocatedItems || currentItems.value
  const experienceStock = row.allocatedExperience != null ? Number(row.allocatedExperience) : bookExperience(currentItems.value)
  if (field === 'level') {
    const req = calculateLevelRequirements(Number(row.level) || 0, target, firstSubProf(row), false)
    const itemsReady = Object.keys(req.items || {}).every(function (id) { return (Number(itemsStock[id]) || 0) >= req.items[id] })
    return itemsReady && Number(req.experience) <= experienceStock
  }
  const req = calculateXiuweiRequirements(Number(row.elite) || 0, target, xiuweiJob(row.prof))
  return Object.keys(req.items || {}).every(function (id) { return (Number(itemsStock[id]) || 0) >= req.items[id] })
}

function growthUpgradeKeyFor(row, field) {
  return row.id + ':' + field
}

function growthUpgradePreview(row, field) {
  return growthUpgradePreviews.value[growthUpgradeKeyFor(row, field)] || null
}

function growthUpgradeError(row, field) {
  return growthUpgradeErrors.value[growthUpgradeKeyFor(row, field)] || ''
}

function growthUpgradeBusy(row, field) {
  return growthUpgradeBusyKeys.value.has(growthUpgradeKeyFor(row, field))
}

function growthUpgradeRequirements(row, field) {
  const preview = growthUpgradePreview(row, field)
  return preview && Array.isArray(preview.requirements) ? preview.requirements : []
}

function growthUpgradeReqValue(req, key) {
  if (!req) return 0
  const camel = key.replace(/_([a-z])/g, function (_, letter) { return letter.toUpperCase() })
  return Number(req[key] != null ? req[key] : req[camel]) || 0
}

function growthUpgradeReqLabel(req) {
  const balance = growthUpgradeReqValue(req, 'balance_after')
  if (balance < 0) return '缺 ' + formatNumber(Math.abs(balance))
  return formatNumber(growthUpgradeReqValue(req, 'owned')) + ' → ' + formatNumber(balance)
}

function growthUpgradeBlocking(row, field) {
  const preview = growthUpgradePreview(row, field)
  const reasons = (preview && (preview.blocking_reasons || preview.blockingReasons)) || []
  return reasons.filter(function (reason) { return reason && reason.code !== 'insufficient_inventory' })
}

function growthUpgradeAvailable(row, field) {
  const preview = growthUpgradePreview(row, field)
  return Boolean(preview && preview.available)
}

function growthUpgradeCanConfirm(row) {
  const field = growthUpgradeField.value
  if (!row || !field || growthUpgradeExecuting.value) return false
  return !growthUpgradeBusy(row, field) && !growthUpgradeError(row, field) && growthUpgradeAvailable(row, field)
}

function openGrowthUpgrade(row, field) {
  if (!row || !row.id || !growthHasUpgrade(row, field) || growthUpgradeExecuting.value || !props.accountId || !trackerGame(row)) return
  growthUpgradeKey.value = row.id
  growthUpgradeField.value = field
  growthUpgradePreviews.value = {}
  growthUpgradeErrors.value = {}
  loadGrowthUpgradePreview(row, field)
}

async function loadGrowthUpgradePreview(row, field) {
  if (!row || !row.id || !props.accountId || !trackerGame(row)) return
  const key = growthUpgradeKeyFor(row, field)
  const seq = (growthUpgradeRequestSeqs.get(key) || 0) + 1
  growthUpgradeRequestSeqs.set(key, seq)
  growthUpgradeBusyKeys.value = new Set(growthUpgradeBusyKeys.value).add(key)
  growthUpgradePreviews.value = Object.assign({}, growthUpgradePreviews.value, { [key]: null })
  growthUpgradeErrors.value = Object.assign({}, growthUpgradeErrors.value, { [key]: '' })
  const entry = currentMap.value[row.id] || row
  try {
    const data = await previewOperatorUpgrade({
      accountId: props.accountId,
      game: trackerGame(row),
      operatorId: row.id,
      dimension: field,
      target: growthStepTarget(row, field),
      expectedOperatorRevision: Number(entry.revision) || 0
    })
    if (growthUpgradeRequestSeqs.get(key) !== seq || growthUpgradeKey.value !== row.id) return
    growthUpgradePreviews.value = Object.assign({}, growthUpgradePreviews.value, { [key]: data })
  } catch (err) {
    if (growthUpgradeRequestSeqs.get(key) !== seq || growthUpgradeKey.value !== row.id) return
    growthUpgradeErrors.value = Object.assign({}, growthUpgradeErrors.value, { [key]: targetErrorMessage(err, '提升预览失败') })
  } finally {
    if (growthUpgradeRequestSeqs.get(key) === seq) {
      const next = new Set(growthUpgradeBusyKeys.value)
      next.delete(key)
      growthUpgradeBusyKeys.value = next
    }
  }
}

function closeGrowthUpgrade() {
  if (growthUpgradeExecuting.value) return
  growthUpgradeKey.value = ''
  growthUpgradeField.value = ''
  growthUpgradePreviews.value = {}
  growthUpgradeErrors.value = {}
}

function uuid() {
  if (globalThis.crypto && typeof globalThis.crypto.randomUUID === 'function') return globalThis.crypto.randomUUID()
  return 'upgrade-' + Date.now() + '-' + Math.random().toString(36).slice(2)
}

function applyUpgradeConsumption(consumed) {
  ;(Array.isArray(consumed) ? consumed : []).forEach(function (item) {
    const type = item.entity_type || item.entityType
    const id = item.id
    const balance = Number(item.balance_after != null ? item.balance_after : item.balanceAfter) || 0
    if (type === 'agent') currentAgents.value = Object.assign({}, currentAgents.value, { [id]: balance })
    else currentItems.value = Object.assign({}, currentItems.value, { [id]: balance })
  })
  const stock = currentItems.value
  currentItems.value = Object.assign({}, stock, {
    __experience__: (Number(stock.bingshucanjuan) || 0) * 100 + (Number(stock.bingshuquanjuan) || 0) * 1000 + (Number(stock.liutaobingshu) || 0) * 10000
  })
}

async function executeGrowthUpgrade(row) {
  const field = growthUpgradeField.value
  if (!growthUpgradeCanConfirm(row)) return
  growthUpgradeExecuting.value = true
  targetError.value = ''
  targetNotice.value = ''
  try {
    const key = growthUpgradeKeyFor(row, field)
    const idempotencyKey = growthUpgradeExecutionKeys.get(key) || uuid()
    growthUpgradeExecutionKeys.set(key, idempotencyKey)
    const entry = currentMap.value[row.id] || row
    const preview = await previewOperatorUpgrade({
      accountId: props.accountId,
      game: trackerGame(row),
      operatorId: row.id,
      dimension: field,
      target: growthStepTarget(row, field),
      expectedOperatorRevision: Number(entry.revision) || 0
    })
    if (!preview || !preview.available) throw new Error('当前状态无法升级，请重新确认')
    const result = await executeOperatorUpgrade({
      accountId: props.accountId,
      game: trackerGame(row),
      operatorId: row.id,
      dimension: field,
      target: growthStepTarget(row, field),
      expectedOperatorRevision: Number(preview.operator_revision != null ? preview.operator_revision : preview.operatorRevision) || 0,
      expectedInventoryRevision: Number(preview.inventory_revision != null ? preview.inventory_revision : preview.inventoryRevision) || 0,
      previewToken: preview.preview_token || preview.previewToken,
      idempotencyKey: idempotencyKey
    })
    applyUpgradeConsumption(result && result.consumed)
    await loadInventory(true)
    growthUpgradeKey.value = ''
    growthUpgradeField.value = ''
    growthUpgradePreviews.value = {}
    growthUpgradeErrors.value = {}
    targetNotice.value = field === 'level' ? '已升级并扣除库存' : '已提升修为并扣除库存'
    if (targetNoticeTimer != null) clearTimeout(targetNoticeTimer)
    targetNoticeTimer = setTimeout(function () { targetNotice.value = '' }, 2000)
    emit('upgrade-applied', { operatorId: row.id })
  } catch (err) {
    targetError.value = targetErrorMessage(err, '升级失败')
  } finally {
    growthUpgradeExecuting.value = false
  }
}

function opGapText(op) {
  const current = currentMap.value[op.id] || {}
  const draft = viewDraft.value[op.id]
  if (!draft) return '未设置目标'
  const level = calculateLevelRequirements(Number(current.level) || 0, Number(draft.level) || 0, firstSubProf(op))
  const xiuwei = calculateXiuweiRequirements(Number(current.elite) || 0, Number(draft.elite) || 0, xiuweiJob(op.prof))
  const net = netRequirement(mergeRequirements(level, xiuwei), currentItems.value)
  const parts = (net.gaps || []).map(function (gap) {
    return itemName(gap.id) + '×' + formatNumber(gap.gap)
  })
  const experienceGap = Math.max(level.experience - bookExperience(currentItems.value), 0)
  if (experienceGap) parts.push('兵书经验×' + formatNumber(experienceGap))
  return parts.length ? parts.join('、') : '当前目标材料已备齐'
}

function growthTargetReached(row) {
  return Number(row.level) >= Number(row.targetLevel) && Number(row.elite) >= Number(row.targetElite)
}

function gapSummary(row) {
  const parts = ((row && row.calculation && row.calculation.gaps) || []).map(function (gap) {
    return itemName(gap.id) + '×' + formatNumber(gap.gap)
  })
  return parts.length ? parts.join('、') : '当前目标材料已备齐'
}

function selectViewOperator(entry) {
  if (!entry || !entry.id) return
  if (viewDraft.value[entry.id]) return
  viewDraft.value = Object.assign({}, viewDraft.value)
  viewDraft.value[entry.id] = viewDraftFor(entry)
  viewSelectedOrder.value = [entry.id].concat(viewSelectedOrder.value.filter(function (item) { return item !== entry.id }))
  viewDirty.value = true
}

function clearViewSelected() {
  viewDraft.value = {}
  viewSelectedOrder.value = []
  viewSavedIds.value = new Set()
  viewCompletedIds.value = new Set()
  viewDirty.value = true
}

function removeViewOperator(id) {
  if (!id || !viewDraft.value[id]) return
  viewDraft.value = Object.assign({}, viewDraft.value)
  delete viewDraft.value[id]
  viewSelectedOrder.value = viewSelectedOrder.value.filter(function (item) { return item !== id })
  const completed = new Set(viewCompletedIds.value)
  completed.delete(id)
  viewCompletedIds.value = completed
  viewDirty.value = true
}

async function deleteViewRow(id) {
  const draft = viewDraft.value[id]
  if (!id || !draft || viewSaving.value || viewSavedIds.value.has(id)) return
  const saved = targets.value[id]
  if (!saved) {
    removeViewOperator(id)
    return
  }
  viewSavedIds.value = new Set(viewSavedIds.value).add(id)
  targetError.value = ''
  targetNotice.value = ''
  try {
    await deleteOperatorGrowthTarget({ accountId: props.accountId, operatorId: id, expectedRevision: Number(saved.revision) || 0 })
    targets.value = Object.assign({}, targets.value)
    delete targets.value[id]
    removeViewOperator(id)
    cacheTargets()
    targetNotice.value = '已移除养成目标'
    if (targetNoticeTimer != null) clearTimeout(targetNoticeTimer)
    targetNoticeTimer = setTimeout(function () { targetNotice.value = '' }, 1800)
  } catch (err) {
    if (err && err.code === 'growth_target_revision_conflict') await loadTargets()
    targetError.value = targetErrorMessage(err, '养成目标删除失败')
  } finally {
    const next = new Set(viewSavedIds.value)
    next.delete(id)
    viewSavedIds.value = next
  }
}

function isViewSelected(id) {
  return Boolean(viewDraft.value[id])
}

function markViewChanged(id) {
  viewDirty.value = true
  if (!id || !viewCompletedIds.value.has(id)) return
  const next = new Set(viewCompletedIds.value)
  next.delete(id)
  viewCompletedIds.value = next
}

function closeViewEditor() {
  viewEditor.value = ''
  viewSaving.value = false
  viewDraft.value = {}
  viewSelectedOrder.value = []
  viewSavedIds.value = new Set()
  viewCompletedIds.value = new Set()
  viewSearch.value = ''
  viewProfFilter.value = 'all'
  viewSubProfFilter.value = 'all'
  viewOwnedOnly.value = false
  viewDirty.value = false
  viewConfirmOpen.value = false
  viewConfirmMode.value = ''
}

async function saveViewTarget(mode) {
  const ids = Object.keys(viewDraft.value)
  if (!ids.length) {
    closeViewEditor()
    return
  }
  if (viewSaving.value) return
  viewSaving.value = true
  targetError.value = ''
  targetNotice.value = ''
  try {
    const results = await Promise.all(ids.map(async function (id) {
      const entry = props.catalogEntries.find(function (item) { return item.id === id })
      const draft = viewDraft.value[id]
      if (!entry || !draft) return null
      return saveOneViewTarget(mode, id, entry, draft)
    }))
    const saved = results.filter(Boolean)
    saved.forEach(function (result) { applyTargetItem(result.id, result.item) })
    cacheTargets()
    targetNotice.value = '已同步 ' + saved.length + ' 位密探的养成目标'
    if (targetNoticeTimer != null) clearTimeout(targetNoticeTimer)
    targetNoticeTimer = setTimeout(function () { targetNotice.value = '' }, 2000)
    closeViewEditor()
  } catch (err) {
    if (err && err.code === 'growth_target_revision_conflict') await loadTargets()
    targetError.value = targetErrorMessage(err, '养成目标保存失败')
  } finally {
    viewSaving.value = false
  }
}

async function saveOneViewTarget(mode, id, entry, draft) {
  const current = currentMap.value[id] || {}
  const row = Object.assign({}, entry, current)
  const currentLevel = Number(row.level) || 0
  const currentElite = Number(row.elite) || 0
  const currentStar = Number(row.starLevel) || 0
  const level = Math.min(100, Math.max(currentLevel, Math.trunc(Number(draft.level) || 0)))
  const elite = Math.min(maxEliteForLevel(level), Math.max(currentElite, Math.trunc(Number(draft.elite) || 0)))
  const starLevel = Math.min(31, Math.max(currentStar, Math.trunc(Number(draft.starLevel) || 0)))
  const body = { expected_revision: Number(targets.value[id] && targets.value[id].revision) || 0 }
  if (mode === 'growth') {
    body.level = level
    body.elite = elite
  } else {
    body.star_level = starLevel
  }
  const item = await putOperatorGrowthTarget({ accountId: props.accountId, operatorId: id, target: body })
  return { id: id, name: entry.name || id, item: item }
}

async function saveViewRow(mode, id) {
  const entry = props.catalogEntries.find(function (item) { return item.id === id })
  const draft = viewDraft.value[id]
  if (!entry || !draft || viewSaving.value) return
  const busy = viewSavedIds.value
  if (busy.has(id)) return
  viewSavedIds.value = new Set(busy).add(id)
  targetError.value = ''
  targetNotice.value = ''
  try {
    const result = await saveOneViewTarget(mode, id, entry, draft)
    applyTargetItem(result.id, result.item)
    cacheTargets()
    viewCompletedIds.value = new Set(viewCompletedIds.value).add(result.id)
    targetNotice.value = (entry.name || id) + '的养成目标已同步'
    if (targetNoticeTimer != null) clearTimeout(targetNoticeTimer)
    targetNoticeTimer = setTimeout(function () { targetNotice.value = '' }, 1800)
  } catch (err) {
    if (err && err.code === 'growth_target_revision_conflict') await loadTargets()
    viewCompletedIds.value = new Set(viewCompletedIds.value)
    viewCompletedIds.value.delete(id)
    targetError.value = targetErrorMessage(err, '养成目标保存失败')
  } finally {
    const next = new Set(viewSavedIds.value)
    next.delete(id)
    viewSavedIds.value = next
  }
}

function starStagesFor(row) {
  const currentStage = starStage(row && row.starLevel)
  return starStages.filter(function (stage) {
    return stage.value === 31 || starStage(stage.value) >= currentStage
  })
}

function flattenCurrent(data) {
  const result = {}
  const rows = Array.isArray(data) ? data : (data ? [data] : [])
  rows.forEach(function (row) {
    const entries = row && row.entries && typeof row.entries === 'object' ? row.entries : {}
    Object.keys(entries).forEach(function (id) {
      const value = entries[id]
      result[id] = Number(value && value.count != null ? value.count : value) || 0
    })
  })
  return result
}

function flattenAcquired(data) {
  const source = data && data.acquired && typeof data.acquired === 'object' ? data.acquired : {}
  const result = {}
  Object.keys(source).forEach(function (id) { result[id] = Number(source[id]) || 0 })
  return result
}

async function loadInventory(forceCurrent) {
  if (!props.isLoggedIn || !props.accountId) {
    inventoryLoadSeq += 1
    currentItems.value = {}
    currentAgents.value = {}
    acquiredItems.value = {}
    acquiredAgents.value = {}
    loading.value = false
    error.value = ''
    return
  }
  const targetAccount = props.accountId
  const seq = ++inventoryLoadSeq
  loading.value = true
  error.value = ''
  try {
    const reuseCurrent = !forceCurrent && props.currentInventoryReady
    const results = await Promise.all([
      reuseCurrent ? Promise.resolve(null) : getCurrent({ accountId: targetAccount, entityType: 'item' }),
      reuseCurrent ? Promise.resolve(null) : getCurrent({ accountId: targetAccount, entityType: 'agent' }),
      getAcquired({ accountId: targetAccount, entityType: 'item', from: rangeFrom, to: rangeTo }),
      getAcquired({ accountId: targetAccount, entityType: 'agent', from: rangeFrom, to: rangeTo })
    ])
    if (seq !== inventoryLoadSeq || props.accountId !== targetAccount) return
    currentItems.value = reuseCurrent ? Object.assign({}, props.initialCurrentItems) : flattenCurrent(results[0])
    currentAgents.value = reuseCurrent ? Object.assign({}, props.initialCurrentAgents) : flattenCurrent(results[1])
    acquiredItems.value = flattenAcquired(results[2])
    acquiredAgents.value = flattenAcquired(results[3])
  } catch (err) {
    if (seq !== inventoryLoadSeq || props.accountId !== targetAccount) return
    error.value = err && err.message ? err.message : '库存或流水加载失败'
  } finally {
    if (seq === inventoryLoadSeq && props.accountId === targetAccount) loading.value = false
  }
}

function firstSubProf(row) { return Array.isArray(row && row.subProf) ? row.subProf[0] : (row && row.subProf ? String(row.subProf).split('、')[0] : '') }
function profIcon(prof) { const file = PROF_ICON_FILES[String(prof || '').split('、')[0]]; return file ? import.meta.env.BASE_URL + 'assets/prof-icons/' + file : '' }
function entryMeta(row) {
  const parts = [String(row && row.prof || '').split('、')[0] || '未知属性']
  const sub = firstSubProf(row)
  if (sub) parts.push(sub)
  return parts.join(' · ')
}
function xiuweiJob(prof) { const first = String(prof || '').split('、')[0]; return ['风', '火'].includes(first) ? 'fh' : ['水', '地'].includes(first) ? 'ds' : 'yy' }
function starStage(level) { return starStageFromLevel(level) }
function starLabel(level) { return Number(level) <= 0 ? '未拥有' : Number(level) >= 25 && Number(level) < 31 ? '五星' : starLabelForStage(starStage(level)) }
function progress(current, target) { const a = Number(current) || 0; const b = Number(target) || 0; return b <= 0 ? 100 : Math.min(100, Math.round(a * 100 / b)) }
function monogram(row) { return Array.from(String(row.name || row.id || '?'))[0] || '?' }
function itemName(id) { return id === '__heart__' ? '心纸' : id === '__experience__' ? '兵书经验' : itemMap.value[id] || id }
function formatNumber(value) { return (Number(value) || 0).toLocaleString('zh-CN') }
function formatMoney(value) { return formatNumber(value) }
function materialSummary(requirement) { return Object.keys(requirement.items || {}).filter(function (id) { return requirement.items[id] > 0 }).slice(0, 3).map(function (id) { return itemName(id) + '×' + formatNumber(requirement.items[id]) }).join('、') }
function bookExperience(stock) {
  return (Number(stock.bingshucanjuan) || 0) * 100 +
    (Number(stock.bingshuquanjuan) || 0) * 1000 +
    (Number(stock.liutaobingshu) || 0) * 10000
}
function rateFor(id) {
  if (id === '__experience__') return bookExperience(acquiredItems.value) / rangeDays
  return (Number(acquiredItems.value[id]) || 0) / rangeDays
}
function rateLabel(id) { const rate = rateFor(id); return rate > 0 ? '日均 ' + rate.toFixed(1) : '暂无速度' }
function heartRateLabel(id) { const rate = (Number(acquiredAgents.value[id]) || 0) / rangeDays; return rate > 0 ? '日均 ' + rate.toFixed(1) : '暂无速度' }
function etaForGaps(gaps) {
  if (!gaps.length) return 0
  const days = gaps.map(function (gap) {
    const rate = gap.id === '__heart__'
      ? (Number(acquiredAgents.value[gap.agentId]) || 0) / rangeDays
      : rateFor(gap.id)
    return rate > 0 ? gap.gap / rate : null
  })
  if (days.some(function (value) { return value == null })) return null
  return Math.max.apply(Math, days)
}
function formatEta(days) { if (days <= 0) return '无需等待'; if (days < 1) return '不足 1 天'; return Math.ceil(days) + ' 天' }

watch(function () { return [props.accountId, props.isLoggedIn, props.refreshKey] }, function () { loadTargets(); loadInventory() }, { immediate: true })
watch(function () { return [props.currentInventoryReady, props.initialCurrentItems, props.initialCurrentAgents] }, function () {
  if (!props.currentInventoryReady) return
  currentItems.value = Object.assign({}, props.initialCurrentItems)
  currentAgents.value = Object.assign({}, props.initialCurrentAgents)
})

onBeforeUnmount(function () {
  if (targetNoticeTimer != null) clearTimeout(targetNoticeTimer)
})
</script>

<style scoped>
.growth-tracker { margin-top: 18px; background: var(--surface); border: 1px solid var(--line); border-radius: 22px; padding: 22px; }
.tracker-heading { display: flex; justify-content: space-between; align-items: flex-start; gap: 16px; padding-bottom: 18px; border-bottom: 1px dashed var(--line); }
.section-kicker { display: block; color: var(--accent-strong); font-size: 11px; font-weight: 800; letter-spacing: 0; }
.tracker-heading h2 { margin-top: 6px; font-family: var(--font-s); font-size: 23px; font-weight: 900; letter-spacing: 0; color: var(--ink); }
.tracker-heading p { margin-top: 5px; color: var(--ink-60); font-size: 12px; line-height: 1.7; }
.tracker-refresh { min-height: 44px; border: 1px solid var(--line); border-radius: 10px; background: var(--paper); color: var(--ink); padding: 0 13px; display: inline-flex; align-items: center; gap: 7px; font: 700 12px var(--font-b); cursor: pointer; }
.tracker-refresh:hover:not(:disabled) { border-color: var(--accent); color: var(--accent-strong); }
.tracker-refresh:focus-visible, .tracker-state button:focus-visible { outline: 2px solid var(--brand-blue); outline-offset: 2px; }
.tracker-refresh:disabled { opacity: .55; cursor: wait; }
.spin { animation: tracker-spin .9s linear infinite; }
@keyframes tracker-spin { to { transform: rotate(360deg); } }
.tracker-overview { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 1px; margin: 18px 0; background: var(--line); border: 1px solid var(--line); border-radius: 14px; overflow: hidden; }
.overview-cell { min-width: 0; background: var(--cream); padding: 14px; }
.overview-cell span { display: block; color: var(--ink-60); font-size: 11px; font-weight: 800; }
.overview-cell strong { display: block; margin-top: 5px; color: var(--accent-strong); font: 900 25px var(--font-d); }
.overview-cell strong small { margin-left: 3px; font: 700 11px var(--font-b); }
.overview-cell em { display: block; margin-top: 3px; color: var(--ink-35); font-size: 11px; font-style: normal; }
.tracker-state { display: flex; align-items: center; justify-content: center; gap: 8px; min-height: 100px; padding: 20px; color: var(--ink-60); font-size: 13px; text-align: center; }
.tracker-state.is-error { color: var(--rouge); background: rgba(166, 81, 74, .07); border-radius: 12px; }
.tracker-state button { border: 0; background: transparent; color: var(--accent-strong); font-weight: 800; text-decoration: underline; cursor: pointer; }
.tracker-state.empty { min-height: 80px; margin-top: 14px; border: 1px dashed var(--line); border-radius: 12px; }
.aggregate-plan { margin: 18px 0 14px; padding: 14px 0; border-top: 1px solid var(--line); border-bottom: 1px solid var(--line); }
.aggregate-head { display: flex; align-items: flex-start; justify-content: space-between; gap: 14px; }
.aggregate-head h3 { color: var(--ink); font-family: var(--font-s); font-size: 14px; font-weight: 900; letter-spacing: 0; }
.aggregate-head p { margin-top: 3px; color: var(--ink-60); font-size: 10.5px; }
.aggregate-head > span { flex: none; color: var(--ink-60); font: 800 11px var(--font-d); }
.aggregate-eta { margin-top: 8px; color: var(--ink-60); font-size: 10.5px; }
.tracker-list { display: flex; flex-direction: column; gap: 14px; }
.tracker-row { border: 1px solid var(--line); border-radius: 15px; background: var(--paper); padding: 15px; }
.tracker-row-head { display: flex; justify-content: space-between; align-items: flex-start; gap: 16px; }
.tracker-identity { display: flex; align-items: center; gap: 10px; min-width: 0; }
.tracker-avatar { position: relative; width: 48px; height: 48px; flex: none; overflow: visible; display: flex; align-items: center; justify-content: center; border: 2px solid var(--line); border-radius: 10px; background: var(--cream); color: var(--ink-35); font: 900 21px var(--font-s); }
.tracker-avatar img { width: 100%; height: 100%; display: block; object-fit: cover; border-radius: 8px; }
.tracker-avatar > span { display: grid; width: 100%; height: 100%; place-items: center; }
.tracker-avatar.rarity-r5 { border-color: var(--accent); }
.tracker-avatar.rarity-r4 { border-color: var(--brand-blue); }
.tracker-identity h3 { overflow: hidden; color: var(--ink); font-size: 15px; font-weight: 900; text-overflow: ellipsis; white-space: nowrap; }
.tracker-identity p { display: flex; align-items: center; flex-wrap: wrap; gap: 4px; margin-top: 3px; color: var(--ink-60); font-size: 11px; }
.tracker-identity p > span + span::before { margin-right: 4px; content: '·'; }
.tracker-prof { display: inline-flex; align-items: center; gap: 3px; }
.tracker-prof img { width: 17px; height: 17px; object-fit: contain; }
.tracker-targets { display: flex; flex-wrap: wrap; justify-content: flex-end; gap: 7px; }
.tracker-target-error { margin: 0 0 8px; color: var(--rouge); font-size: 10px; font-weight: 700; text-align: right; }
.tracker-target-notice { margin: 0 0 8px; color: var(--accent-strong); font-size: 10px; font-weight: 700; text-align: right; }
.tracker-targets label { display: inline-flex; align-items: center; gap: 5px; color: var(--ink-60); font-size: 11px; font-weight: 800; }
.tracker-targets input, .tracker-targets select { width: 58px; min-height: 32px; border: 1px solid var(--line); border-radius: 7px; background: var(--surface); color: var(--ink); padding: 4px 6px; font: 800 12px var(--font-d); outline: none; }
.tracker-targets select { width: 92px; font-family: var(--font-b); }
.tracker-targets input:focus, .tracker-targets select:focus { border-color: var(--accent); }
.tracker-targets input:focus-visible, .tracker-targets select:focus-visible { outline: 2px solid var(--brand-blue); outline-offset: 1px; }
.tracker-targets input:disabled, .tracker-targets select:disabled { opacity: .58; cursor: wait; }
.tracker-progress-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 12px; margin-top: 15px; }
.progress-block { min-width: 0; }
.progress-title { display: flex; justify-content: space-between; gap: 8px; color: var(--ink-60); font-size: 11px; font-weight: 800; }
.progress-title b { color: var(--ink); font-family: var(--font-d); white-space: nowrap; }
.progress-track { height: 7px; margin-top: 7px; overflow: hidden; border-radius: 99px; background: var(--cream); }
.progress-track i { display: block; height: 100%; min-width: 2px; border-radius: inherit; background: var(--accent); transition: width .35s var(--ease); }
.progress-track.mint i { background: #BFDCC0; }
.progress-track.rose i { background: var(--rouge); }
.progress-block p { min-height: 32px; margin-top: 6px; color: var(--ink-60); font-size: 10.5px; line-height: 1.55; }
.progress-block p strong { color: var(--rouge); font-family: var(--font-d); }
.tracker-materials { margin-top: 12px; padding-top: 12px; border-top: 1px dashed var(--line); }
.materials-head { display: flex; justify-content: space-between; gap: 8px; color: var(--ink); font-size: 12px; font-weight: 900; }
.materials-head small { color: var(--ink-35); font-size: 10.5px; font-weight: 600; }
.materials-clear { margin-top: 8px; color: var(--ink); font-size: 11px; font-weight: 800; }
.material-chips { display: flex; flex-wrap: wrap; gap: 7px; margin-top: 8px; }
.material-chip { display: inline-flex; align-items: baseline; gap: 5px; min-height: 28px; border: 1px solid rgba(215, 137, 53, .35); border-radius: 8px; background: var(--cream); padding: 5px 8px; }
.material-chip b { color: var(--ink); font-size: 11px; }
.material-chip em { color: var(--rouge); font: 800 11px var(--font-d); font-style: normal; }
.material-chip small { color: var(--ink-35); font-size: 10px; }
.material-chip.heart-chip { border-color: rgba(166, 81, 74, .35); }
.eta-line { display: flex; justify-content: space-between; gap: 10px; margin-top: 9px; color: var(--ink-60); font-size: 10.5px; }
.money-total { color: var(--ink-35); white-space: nowrap; }
.tracker-mode-switch { display: inline-flex; gap: 4px; padding: 4px; margin-top: 14px; border: 1px solid var(--line); border-radius: 10px; background: rgba(73, 59, 44, .06) }
.tracker-mode-switch button { min-height: 32px; padding: 5px 12px; border: 0; border-radius: 7px; background: transparent; color: var(--ink-60); font: 800 11px var(--font-b); cursor: pointer }
.tracker-mode-switch button:hover { color: var(--ink) }
.tracker-mode-switch button.on { background: var(--surface); color: var(--accent-strong); box-shadow: 0 1px 4px rgba(73, 59, 44, .16) }
.tracker-mode-switch button:focus-visible { outline: 2px solid var(--brand-blue); outline-offset: 1px }
.tracker-mode-body { margin-top: 14px }
.tracker-mode-summary { display: flex; align-items: baseline; gap: 6px; color: var(--ink-60); font-size: 11px; font-weight: 800 }
.tracker-mode-summary strong { color: var(--accent-strong); font: 900 16px var(--font-d) }
.tracker-mode-actions { display: flex; align-items: center; gap: 8px; margin-left: auto }
.tracker-edit-button { display: inline-flex; min-height: 30px; align-items: center; gap: 4px; padding: 4px 10px; border: 1px solid var(--line); border-radius: 7px; background: var(--surface); color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.tracker-edit-button:hover { border-color: var(--accent); color: var(--accent-strong) }
.tracker-edit-button.on { border-color: var(--accent); background: rgba(166, 81, 74, .07); color: var(--accent-strong) }
.tracker-edit-button:focus-visible { outline: 2px solid var(--brand-blue); outline-offset: 1px }
.growth-allocation-panel { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 10px; padding: 10px; border: 1px solid var(--line); border-radius: 10px; background: var(--cream) }
.growth-allocation-head { display: flex; flex: 1 1 100%; align-items: center; justify-content: space-between; gap: 8px }
.allocation-mode { display: inline-flex; gap: 3px; padding: 3px; border: 1px solid var(--line); border-radius: 8px; background: var(--surface) }
.allocation-mode button { min-height: 28px; padding: 4px 12px; border: 0; border-radius: 6px; background: transparent; color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.allocation-mode button.on { background: var(--paper); color: var(--accent-strong); box-shadow: 0 1px 4px rgba(73, 59, 44, .16) }
.allocation-reset { min-height: 28px; padding: 4px 10px; border: 1px solid var(--line); border-radius: 6px; background: var(--paper); color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.allocation-reset:hover:not(:disabled) { border-color: var(--accent); color: var(--accent-strong) }
.allocation-reset:disabled { opacity: .5; cursor: not-allowed }
.allocation-hint { flex: 1 1 100%; margin: 0; color: var(--ink-60); font-size: 10px; font-weight: 700 }
.growth-allocation-list { display: flex; flex: 1 1 100%; flex-direction: column; gap: 6px }
.growth-allocation-row { display: grid; grid-template-columns: 24px 22px minmax(0, 1fr) auto auto 74px; align-items: center; gap: 8px; min-width: 0; padding: 6px 9px; border: 1px solid var(--line); border-radius: 8px; background: var(--paper); cursor: grab }
.growth-allocation-row.ready { border-color: rgba(111, 159, 118, .45); background: rgba(191, 220, 192, .22) }
.growth-allocation-row.dragging { opacity: .5; border-color: var(--accent); box-shadow: 0 6px 14px rgba(73, 59, 44, .14) }
.allocation-rank { color: var(--ink-35); font-family: var(--font-d); font-weight: 800; text-align: center }
.allocation-drag { color: var(--ink-35); font-size: 12px; line-height: 1; text-align: center; user-select: none }
.allocation-unowned, .selected-unowned { display: inline-flex; align-items: center; padding: 1px 6px; border-radius: 99px; background: rgba(166, 81, 74, .08); color: #b44840; font-size: 9px; font-weight: 800; line-height: 1.5 }
.selected-unowned { margin-left: 2px }
.allocation-name { overflow: hidden; color: var(--ink); font: 800 10.5px var(--font-b); text-overflow: ellipsis; white-space: nowrap }
.allocation-rarity { color: var(--ink-60); font-size: 10px; font-weight: 800 }
.allocation-status { color: var(--ink-35); font-size: 10px; font-weight: 700; text-align: right }
.growth-allocation-row.ready .allocation-status { color: #3f6b46; font-weight: 900 }
.allocation-actions { display: inline-flex; justify-content: flex-end; gap: 4px }
.allocation-actions button { width: 22px; height: 22px; padding: 0; border: 1px solid var(--line); border-radius: 5px; background: var(--surface); color: var(--ink-60); font-size: 11px; cursor: pointer }
.allocation-actions button:hover:not(:disabled) { border-color: var(--accent); color: var(--accent-strong) }
.allocation-actions button:disabled { opacity: .4; cursor: not-allowed }
.tracker-view-editor select, .tracker-view-editor input { min-height: 32px; padding: 4px 7px; border: 1px solid var(--line); border-radius: 6px; background: var(--surface); color: var(--ink); font: 800 11px var(--font-b); outline: none }
.tracker-view-editor select:focus, .tracker-view-editor input:focus { border-color: var(--accent) }
.tracker-view-editor { display: flex; flex-wrap: wrap; align-items: stretch; gap: 8px; margin-top: 10px; padding: 10px; border: 1px solid var(--line); border-radius: 10px; background: var(--cream) }
.tracker-view-confirm { position: fixed; z-index: 60; inset: 0; display: flex; align-items: center; justify-content: center; padding: 18px; background: rgba(73, 59, 44, .32) }
.tracker-view-confirm-box { width: min(320px, 100%); padding: 16px; border: 1px solid var(--line); border-radius: 12px; background: var(--paper); box-shadow: 0 18px 44px rgba(73, 59, 44, .26) }
.tracker-view-confirm-box h3 { margin: 0 0 4px; color: var(--ink); font: 900 14px var(--font-b) }
.tracker-view-confirm-box p { margin: 0 0 12px; color: var(--ink-60); font-size: 10.5px; line-height: 1.5 }
.tracker-view-confirm-actions { display: flex; justify-content: flex-end; gap: 8px }
.tracker-view-confirm-actions button { min-height: 32px; padding: 5px 14px; border: 1px solid var(--line); border-radius: 7px; background: var(--surface); color: var(--ink-60); font: 800 10.5px var(--font-b); cursor: pointer }
.tracker-view-confirm-actions button:hover:not(:disabled) { border-color: var(--accent); color: var(--accent-strong) }
.tracker-view-confirm-actions button:first-child { border-color: rgba(111, 159, 118, .5); background: #f4f8f0; color: #3f6b46 }
.tracker-view-confirm-actions button:disabled { opacity: .55; cursor: wait }
.tracker-view-search { display: flex; flex: 1 1 100%; gap: 6px }
.tracker-view-search input[type="search"] { flex: 1 1 260px; min-width: 180px }
.tracker-owned-only { position: relative; display: inline-flex; min-height: 32px; align-items: center; gap: 6px; padding: 0 10px; border: 1px solid var(--line); border-radius: 99px; background: var(--paper); color: var(--ink-60); font: 700 10px var(--font-b); white-space: nowrap; cursor: pointer; transition: border-color .2s ease, background-color .2s ease, color .2s ease }
.tracker-owned-only input { position: absolute; width: 1px; height: 1px; opacity: 0; pointer-events: none }
.tracker-owned-check { position: relative; flex: none; display: block; width: 14px; height: 14px; border: 1px solid var(--line); border-radius: 4px; background: var(--surface); transition: border-color .2s ease, background-color .2s ease }
.tracker-owned-check i { position: absolute; left: 4px; top: 1px; display: none; width: 5px; height: 9px; border: solid #fff; border-width: 0 2px 2px 0; transform: rotate(45deg) }
.tracker-owned-only.on { border-color: rgba(166, 81, 74, .4); background: rgba(166, 81, 74, .08); color: var(--accent-strong) }
.tracker-owned-only.on .tracker-owned-check { border-color: var(--accent-strong); background: var(--accent-strong) }
.tracker-owned-only.on .tracker-owned-check i { display: block }
.view-search-clear { flex: none; width: 32px; min-width: 32px; padding: 0 !important; border: 1px solid var(--line) !important; border-radius: 6px !important; background: var(--surface) !important; color: var(--ink-60) !important; font: 800 16px var(--font-b) !important; line-height: 1; cursor: pointer }
.tracker-view-filters { display: flex; flex: 1 1 100%; flex-direction: column; gap: 6px }
.tracker-view-filters .pf-row { display: flex; align-items: center; gap: 6px; flex-wrap: wrap; min-width: 0 }
.tracker-view-filters .pf-label { min-width: 0; font-size: 10px; white-space: nowrap }
.tracker-view-filters .mf-filter { padding: 3px; gap: 3px; border-radius: 8px }
.tracker-view-filters .mf-filter button { display: inline-flex; min-height: 28px; align-items: center; gap: 4px; padding: 4px 9px; border: 0; border-radius: 6px; background: transparent; color: var(--ink-60); font: 700 11px var(--font-b); cursor: pointer }
.tracker-view-filters .mf-filter button:hover:not(.on) { color: var(--ink) }
.tracker-view-filters .mf-filter button.on { background: var(--surface); color: var(--accent-strong); box-shadow: 0 1px 4px rgba(73, 59, 44, .16) }
.tracker-view-filters .mf-filter img { display: block; width: 16px; height: 16px; object-fit: contain; vertical-align: middle }
.tracker-view-selected { display: flex; flex: 1 1 100%; flex-wrap: wrap; align-items: center; gap: 9px; padding: 8px 9px; border: 1px solid var(--accent); border-radius: 9px; background: var(--surface) }
.tracker-view-selected-head { display: flex; flex: 1 1 100%; align-items: center; justify-content: space-between; gap: 8px }
.tracker-view-selected-head strong { color: var(--accent-strong); font: 900 12px var(--font-b) }
.tracker-view-clear { min-height: 26px; padding: 2px 10px; border: 1px solid var(--line); border-radius: 6px; background: var(--paper); color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.tracker-view-clear:hover { border-color: var(--rouge); color: var(--rouge) }
.tracker-view-selected .tracker-avatar { width: 38px; height: 38px; border-radius: 9px; font-size: 16px }
.tracker-view-selected-name { min-width: 110px; flex: 1 }
.tracker-view-selected-name h3 { overflow: hidden; color: var(--ink); font-size: 12px; font-weight: 900; text-overflow: ellipsis; white-space: nowrap }
.tracker-view-selected-name p { display: flex; flex-wrap: wrap; gap: 3px 6px; margin-top: 2px; color: var(--ink-60); font-size: 10px }
.tracker-view-edit-row { display: flex; flex: 1 1 100%; flex-wrap: wrap; align-items: center; gap: 8px; min-width: 0; padding: 7px 9px; border: 1px solid var(--line); border-radius: 8px; background: var(--paper) }
.tracker-view-edit-head { display: flex; min-width: 150px; flex: 1; align-items: center; gap: 8px }
.tracker-view-edit-head .tracker-avatar { width: 38px; height: 38px; border-radius: 9px; font-size: 16px }
.tracker-compare { display: grid; grid-template-columns: 52px 1fr 1fr; gap: 3px 8px; min-width: 170px; width: 210px; padding: 6px 8px; border: 1px solid var(--line); border-radius: 7px; background: var(--cream); font-size: 10px; font-weight: 700 }
.tracker-compare-row { display: contents }
.tracker-compare-row span { color: var(--ink-60) }
.tracker-compare-row b { color: var(--ink); font-family: var(--font-d); text-align: right }
.tracker-compare-row.head b { color: var(--accent-strong); font-family: var(--font-b); font-size: 9px }
.tracker-compare-toggle { grid-column: 1 / -1; min-height: 24px; margin-top: 3px; padding: 3px 8px; border: 1px solid var(--line); border-radius: 6px; background: var(--surface); color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.tracker-compare-toggle:hover { color: var(--accent-strong); border-color: var(--accent) }
.tracker-compare-gap { grid-column: 1 / -1; padding: 5px 7px; border-radius: 6px; background: var(--paper); color: var(--ink); font-size: 10px; line-height: 1.55 }
.tracker-view-row-actions { display: inline-flex; flex: none; align-items: center; gap: 5px }
.tracker-view-row-actions button { min-height: 28px; padding: 3px 10px; border-radius: 6px; font: 800 10px var(--font-b); cursor: pointer }
.tracker-view-row-actions .row-delete { border: 1px solid rgba(166, 81, 74, .3); background: var(--cream); color: var(--rouge) }
.tracker-view-row-actions .row-delete:hover { background: var(--rouge); color: #fff }
.tracker-view-results { display: grid; flex: 1 1 100%; grid-template-columns: repeat(auto-fill, minmax(76px, 1fr)); gap: 7px; max-height: 300px; overflow-y: auto }
.groups-panel { flex: 1 1 100%; margin-top: 12px; padding: 10px; border: 1px solid var(--line); border-radius: 10px; background: var(--surface) }
.editor-groups { flex: 1 1 100%; margin-top: 10px; padding: 9px; border: 1px solid var(--line); border-radius: 9px; background: var(--surface) }
.editor-groups-head { display: flex; flex-wrap: wrap; align-items: center; justify-content: space-between; gap: 7px }
.editor-groups-head h3 { color: var(--ink); font: 900 12px var(--font-b) }
.editor-groups-hint { color: var(--ink-60); font-size: 10px; font-weight: 700 }
.editor-groups-list { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 8px }
.editor-group-chip { display: inline-flex; align-items: center; gap: 5px; max-width: 100%; padding: 4px 7px; border: 1px solid var(--line); border-radius: 7px; background: var(--paper); cursor: pointer; font-size: 10.5px }
.editor-group-chip.active { border-color: var(--accent); box-shadow: inset 0 0 0 1px rgba(166, 81, 74, .26); background: var(--cream) }
.editor-group-chip b { color: var(--ink); font-weight: 900 }
.editor-group-chip em { color: var(--ink-60); font-style: normal; font-weight: 800 }
.editor-group-chip input { min-width: 54px; width: 64px; padding: 2px 4px; border: 1px solid var(--line); border-radius: 5px; background: var(--surface); color: var(--ink); font: 700 10.5px var(--font-b) }
.editor-group-chip label { display: inline-flex; align-items: center; gap: 3px; color: var(--ink-60); font: 700 10px var(--font-b) }
.editor-group-chip label input { width: auto; min-width: 0 }
.editor-group-chip button { width: 20px; height: 20px; padding: 0; border: 0; border-radius: 5px; background: rgba(166, 81, 74, .12); color: var(--rouge); font: 800 10px var(--font-b); cursor: pointer }
.editor-group-members { display: flex; flex-wrap: wrap; gap: 5px; margin-top: 8px }
.editor-groups-stats { flex: 1 1 100%; margin-top: 10px; padding: 9px; border: 1px solid var(--line); border-radius: 9px; background: var(--surface) }
.group-stats-page { flex: 1 1 100%; margin-top: 14px; padding: 11px; border: 1px solid var(--line); border-radius: 11px; background: var(--surface) }
.group-stats-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 9px; margin-top: 9px }
.group-stats-card { display: flex; flex-direction: column; gap: 8px; min-width: 0; padding: 10px; border: 1px solid var(--line); border-radius: 9px; background: var(--paper) }
.group-stats-head { display: flex; align-items: center; justify-content: space-between; gap: 8px }
.group-stats-head h4 { color: var(--ink); font: 900 13px var(--font-b) }
.group-stats-head span { color: var(--ink-60); font: 800 10px var(--font-b) }
.group-stats-union-check { display: inline-flex; align-items: center; gap: 4px; color: var(--ink-60); font: 700 10px var(--font-b); cursor: pointer }
.group-stats-union-check input { accent-color: var(--accent-strong) }
.group-stats-name-input { min-width: 70px; flex: 1; padding: 2px 5px; border: 1px solid var(--line); border-radius: 5px; background: var(--cream); color: var(--ink); font: 900 12px var(--font-b) }
.group-stats-controls { display: inline-flex; gap: 3px; opacity: 0; transition: opacity .15s var(--ease) }
.group-stats-card:hover .group-stats-controls, .group-stats-controls:focus-within { opacity: 1 }
.group-stats-controls button { width: 20px; height: 20px; padding: 0; border: 1px solid var(--line); border-radius: 5px; background: var(--cream); color: var(--ink-60); font-size: 11px; cursor: pointer }
.group-stats-controls button:hover { color: var(--accent-strong); border-color: var(--accent) }
.group-stats-rows { display: flex; flex-direction: column; gap: 5px }
.group-stats-row { display: flex; flex-wrap: wrap; align-items: baseline; gap: 5px 8px; padding: 5px 7px; border: 1px solid var(--line); border-radius: 7px; background: var(--cream) }
.group-stats-row b { color: var(--ink); font-weight: 900 }
.group-stats-row em { color: var(--accent-strong); font: 800 10px var(--font-d); font-style: normal }
.group-stats-row small { color: var(--ink-60); font-weight: 700 }
.group-stats-union { margin-top: 10px; padding: 9px; border: 1px solid var(--line); border-radius: 9px; background: var(--paper) }
.groups-manage-head { display: flex; flex-wrap: wrap; align-items: center; justify-content: space-between; gap: 8px }
.groups-manage-head h3 { color: var(--ink); font: 900 13px var(--font-b) }
.group-create { display: inline-flex; gap: 6px }
.group-create input { min-height: 30px; width: 190px; padding: 4px 8px; border: 1px solid var(--line); border-radius: 7px; background: var(--paper); color: var(--ink); font: 700 11px var(--font-b) }
.group-create button { min-height: 30px; padding: 4px 12px; border: 0; border-radius: 7px; background: var(--accent); color: #fff; font: 800 11px var(--font-b); cursor: pointer }
.groups-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 8px; margin-top: 10px }
.group-card { display: flex; flex-direction: column; gap: 7px; min-width: 0; padding: 9px; border: 1px solid var(--line); border-radius: 9px; background: var(--paper) }
.group-card.active { border-color: var(--accent); box-shadow: inset 0 0 0 1px rgba(166, 81, 74, .3) }
.group-card-head { display: flex; align-items: center; justify-content: space-between; gap: 6px }
.group-card-head span { color: var(--ink-60); font: 800 10px var(--font-b) }
.group-name-input { min-width: 0; flex: 1; padding: 3px 6px; border: 1px solid transparent; border-radius: 6px; background: transparent; color: var(--ink); font: 900 12px var(--font-b) }
.group-name-input:hover, .group-name-input:focus { border-color: var(--line); background: var(--cream) }
.group-check { display: flex; align-items: center; gap: 5px; color: var(--ink-60); font: 700 10px var(--font-b) }
.group-check input { accent-color: var(--accent-strong) }
.group-card-actions { display: flex; gap: 6px }
.group-card-actions button { min-height: 26px; padding: 3px 9px; border: 1px solid var(--line); border-radius: 6px; background: var(--cream); color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.group-card-actions button:hover { color: var(--accent-strong); border-color: var(--accent) }
.group-card-actions .group-delete { color: var(--rouge) }
.group-members { display: flex; flex-wrap: wrap; gap: 5px }
.group-member-chip { display: inline-flex; align-items: center; gap: 4px; max-width: 100%; padding: 3px 6px; border: 1px solid var(--line); border-radius: 6px; background: var(--cream); color: var(--ink); font: 700 10px var(--font-b) }
.group-member-chip em { color: var(--accent-strong); font: 800 10px var(--font-d); font-style: normal }
.group-member-chip button { flex: none; width: 16px; height: 16px; padding: 0; border: 0; border-radius: 50%; background: rgba(166, 81, 74, .15); color: var(--rouge); font-size: 12px; cursor: pointer }
.group-members-empty { color: var(--ink-35); font-size: 10px }
.group-picker, .groups-combined, .all-groups-board { flex: 1 1 100%; margin-top: 12px; padding: 10px; border: 1px solid var(--line); border-radius: 10px; background: var(--surface) }
.groups-combined, .all-groups-board { margin-top: 12px }
.group-union-rows { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 9px }
.group-union-row { display: inline-flex; align-items: center; gap: 6px; max-width: 100%; padding: 5px 8px; border: 1px solid var(--line); border-radius: 7px; background: var(--paper); font-size: 10.5px }
.group-union-row b { color: var(--ink); font-weight: 900 }
.group-union-row em { color: var(--accent-strong); font: 800 10px var(--font-d); font-style: normal }
.group-union-row small { color: var(--ink-60); font-weight: 700 }
.tracker-view-result { display: flex; min-width: 0; flex-direction: column; align-items: center; gap: 3px; padding: 6px 4px; border: 1px solid var(--line); border-radius: 8px; background: var(--paper); color: var(--ink); cursor: pointer }
.tracker-view-result:hover { border-color: var(--accent); background: var(--surface) }
.tracker-view-result.on { border-color: #f6edd0; box-shadow: inset 0 0 0 2px rgba(166, 81, 74, .34); background: #f6edd0 }
.tracker-view-result .tracker-avatar { position: relative }
.tracker-view-result i { position: absolute; right: 3px; bottom: 3px; display: grid; width: 16px; height: 16px; place-items: center; border-radius: 50%; background: var(--accent-strong); color: #fff; font-size: 10px; font-style: normal; line-height: 1 }
.tracker-view-result .tracker-avatar { width: 42px; height: 42px; border-radius: 9px; font-size: 17px }
.tracker-view-result b { overflow: hidden; width: 100%; font: 800 10px var(--font-b); text-align: center; text-overflow: ellipsis; white-space: nowrap }
.tracker-view-result small { overflow: hidden; width: 100%; color: var(--ink-35); font-size: 9px; text-align: center; text-overflow: ellipsis; white-space: nowrap }
.tracker-view-empty { grid-column: 1 / -1; margin: 8px 0; color: var(--ink-35); font: 700 11px var(--font-b); text-align: center }
.tracker-view-actions { display: flex; flex: 1 1 100%; justify-content: flex-end; gap: 7px }
.tracker-view-actions button { min-height: 32px; padding: 4px 12px; border: 0; border-radius: 6px; background: var(--accent); color: #fff; font: 800 11px var(--font-b); cursor: pointer }
.tracker-view-actions button:disabled { opacity: .5; cursor: not-allowed }
.tracker-view-actions button.cancel { background: var(--paper); color: var(--ink-60) }
.tracker-view-actions .tracker-view-save-all { padding: 4px 16px }
.tracker-view-editor label { display: inline-flex; min-height: 32px; align-items: center; gap: 5px; color: var(--ink-60); font-size: 10px; font-weight: 800 }
.tracker-view-editor label select, .tracker-view-editor label input { min-width: 74px; width: 74px }
.tracker-view-editor label .tracker-current-star { display: inline-flex; align-items: center; gap: 3px; color: var(--ink) }
.tracker-view-editor label .tracker-current-star b { color: var(--accent-strong); font-family: var(--font-d); font-weight: 900 }
.growth-compact-list { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 10px; margin-top: 10px; align-items: start }
.growth-compact-row { position: relative; display: flex; flex-direction: column; gap: 9px; min-width: 0; height: auto; padding: 12px; border: 1px solid var(--line); border-top: 2px solid var(--brand-blue); border-radius: 12px; background: var(--paper); box-shadow: 0 6px 16px rgba(73, 59, 44, .08) }
.growth-compact-row > .growth-compact-head { flex: none }
.growth-compact-row.rarity-r5 { border-top-color: var(--accent) }
.growth-compact-row.rarity-r4 { border-top-color: #8672b2 }
.growth-compact-head { display: flex; align-items: center; gap: 9px; min-width: 0 }
.growth-compact-head .tracker-avatar { flex: none; width: 44px; height: 44px; border-radius: 10px; font-size: 17px }
.growth-compact-title { min-width: 0; flex: 1 }
.growth-compact-title h3 { margin: 0; overflow: hidden; color: var(--ink); font: 900 13px/1.3 var(--font-b); text-overflow: ellipsis; white-space: nowrap }
.growth-compact-meta { display: flex; align-items: center; gap: 4px; margin: 3px 0 12px; color: var(--ink-60); font: 700 10px/1.4 var(--font-b) }
.growth-compact-meta img { flex: none; width: 15px; height: 15px; object-fit: contain }
.growth-compare { display: grid; grid-template-columns: 38px 1fr 1fr; gap: 5px 8px; align-items: center; margin-top: 8px; padding: 9px 8px 7px; border: 1px solid var(--line); border-radius: 8px; background: var(--cream) }
.growth-compare-row { display: contents }
.growth-compare-row span, .growth-compare-row b { min-height: 18px; font-size: 10px; line-height: 1.4 }
.growth-compare-row.pair-row { grid-column: 1 / -1; display: grid; grid-template-columns: 38px 1fr auto 1fr; align-items: center; gap: 5px 8px }
.growth-compare-row.pair-row i { color: var(--ink-35); font: 700 10px/1 var(--font-d); font-style: normal; text-align: center }
.growth-compare-row span { align-self: center; color: var(--ink-60); font-weight: 800; text-align: left }
.growth-compare-row b { color: var(--ink); font-family: var(--font-d); font-weight: 800; text-align: center }
.growth-compare-row.head b { color: var(--accent-strong); font-family: var(--font-b); font-weight: 800 }
.growth-compare-bar { grid-column: 1 / -1; height: 4px; margin: -1px 0 2px; overflow: hidden; border-radius: 99px; background: rgba(73, 59, 44, .12) }
.growth-compare-bar i { display: block; height: 100%; border-radius: inherit; background: var(--brand-blue); transition: width .25s ease }
.growth-compare-bar.done i { background: #5d8a5d }
.growth-compare-gap { grid-column: 1 / -1; display: flex; flex-wrap: wrap; gap: 4px 8px; align-items: flex-start; align-content: flex-start; min-height: 80px; max-height: 80px; overflow-y: auto; overflow-x: hidden; scrollbar-width: thin; scrollbar-color: rgba(73, 59, 44, .3) transparent; padding: 5px 4px 0 0; margin-top: 2px; border-top: 1px dashed rgba(73, 59, 44, .22); color: var(--ink-60); font: 700 10px/1.5 var(--font-b) }
.growth-compare-gap::-webkit-scrollbar { width: 5px }
.growth-compare-gap::-webkit-scrollbar-thumb { background: rgba(73, 59, 44, .32); border-radius: 99px }
.growth-compare-gap b { display: block; flex: 1 0 100%; width: 100%; color: var(--ink); font-weight: 800; word-break: break-word }
.growth-gap-ready { display: inline-flex; min-width: 96px; min-height: 28px; align-items: center; justify-content: center; border: 1px solid var(--line); border-radius: 7px; background: rgba(73, 59, 44, .06); color: var(--ink-60); font: 800 10px var(--font-b); font-style: normal }
.growth-gap-ready.is-unowned { color: #b44840; border-color: rgba(166, 81, 74, .38) }
.growth-upgrade-bar { grid-column: 1 / -1; display: flex; min-height: 34px; align-items: center; justify-content: center; gap: 8px; margin-top: 1px; padding-top: 6px; border-top: 1px dashed rgba(73, 59, 44, .16) }
.growth-upgrade-button { min-height: 26px; padding: 3px 12px; border: 1px solid var(--line); border-radius: 7px; background: var(--surface); color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.growth-upgrade-button.ready { border-color: rgba(111, 159, 118, .5); background: #f4f8f0; color: #3f6b46 }
.growth-upgrade-button:hover:not(:disabled) { border-color: var(--accent); color: var(--accent-strong); background: var(--cream) }
.growth-upgrade-button:disabled { opacity: .55; cursor: wait }
.growth-upgrade-pop { position: absolute; z-index: 10; left: 7px; right: 7px; top: calc(100% - 4px); padding: 9px; border: 1px solid var(--line); border-radius: 9px; background: var(--paper); box-shadow: 0 10px 28px rgba(73, 59, 44, .22) }
.growth-upgrade-pop-head { display: flex; align-items: center; justify-content: space-between; gap: 8px; margin-bottom: 6px; color: var(--ink); font: 900 11px var(--font-b) }
.growth-upgrade-pop-head button { flex: none; width: 22px; height: 22px; padding: 0; border: 1px solid var(--line); border-radius: 6px; background: var(--cream); color: var(--ink-60); font-size: 13px; line-height: 1; cursor: pointer }
.growth-upgrade-pop-head button:hover:not(:disabled) { border-color: var(--rouge); color: var(--rouge) }
.growth-upgrade-pop-row { padding: 6px 0; border-top: 1px dashed rgba(73, 59, 44, .16) }
.growth-upgrade-pop-title { display: flex; align-items: center; gap: 6px; color: var(--ink-60); font-size: 10px; font-weight: 700 }
.growth-upgrade-pop-title b { color: var(--ink); font: 900 10px var(--font-b) }
.growth-upgrade-pop-title span { color: var(--accent-strong); font-family: var(--font-d); font-weight: 900 }
.growth-upgrade-pop-state, .growth-upgrade-pop-error { margin-top: 5px; color: var(--ink-60); font-size: 9.5px }
.growth-upgrade-pop-error { color: var(--rouge); font-weight: 700 }
.growth-upgrade-pop-mats { display: flex; flex-wrap: wrap; gap: 4px 7px; margin-top: 5px }
.growth-upgrade-pop-mats span { display: inline-flex; align-items: baseline; gap: 4px; padding: 2px 5px; border: 1px solid var(--line); border-radius: 5px; background: var(--cream); color: var(--ink); font-size: 9.5px; font-weight: 700 }
.growth-upgrade-pop-mats span.is-lack { border-color: rgba(166, 81, 74, .4); color: var(--rouge) }
.growth-upgrade-pop-mats small { color: var(--ink-60); font-weight: 700 }
.growth-upgrade-pop-blocked { display: block; margin-top: 4px; color: var(--rouge); font: 700 9.5px var(--font-b) }
.growth-upgrade-pop-note { display: block; margin-top: 4px; color: var(--ink-60); font: italic 600 12px/1.45 var(--font-b) }
.growth-upgrade-pop-actions { display: flex; justify-content: flex-end; gap: 6px; margin-top: 8px; padding-top: 7px; border-top: 1px dashed rgba(73, 59, 44, .16) }
.growth-upgrade-pop-actions button { min-height: 28px; padding: 4px 10px; border: 1px solid var(--line); border-radius: 6px; background: var(--paper); color: var(--ink-60); font: 800 10px var(--font-b); cursor: pointer }
.growth-upgrade-pop-actions .growth-upgrade-pop-confirm { border-color: rgba(111, 159, 118, .5); background: #f4f8f0; color: #3f6b46 }
.growth-upgrade-pop-actions .growth-upgrade-pop-confirm:hover:not(:disabled) { border-color: var(--accent); color: var(--accent-strong); background: var(--cream) }
.growth-upgrade-pop-actions button:disabled { opacity: .5; cursor: not-allowed }
.growth-clear { color: var(--ink-60); font-size: 10px; font-weight: 800 }
.heart-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(150px, 1fr)); gap: 8px; margin-top: 10px }
.heart-cell { display: flex; flex-direction: column; gap: 8px; min-width: 0; padding: 10px; border: 1px solid var(--line); border-radius: 12px; background: var(--paper) }
.heart-cell-head { display: flex; align-items: center; gap: 8px; min-width: 0 }
.heart-cell-head .tracker-avatar { width: 40px; height: 40px; font-size: 17px }
.heart-cell-head h3 { overflow: hidden; color: var(--ink); font-size: 12px; font-weight: 900; text-overflow: ellipsis; white-space: nowrap }
.heart-cell-head p { margin-top: 2px; color: var(--ink-60); font-size: 10px }
.heart-cell-counts { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 5px }
.heart-cell-counts span { color: var(--ink-60); font-size: 10px; font-weight: 800 }
.heart-cell-counts b { color: var(--ink); font-family: var(--font-d) }
.heart-cell-counts em { color: var(--accent-strong); font-style: normal; font-weight: 900 }
.heart-cell-counts .is-lack b { color: var(--rouge) }
@media (max-width: 760px) {
  .growth-tracker { padding: 15px; border-radius: 17px; }
  .tracker-heading { flex-direction: column; }
  .tracker-refresh { width: 100%; justify-content: center; }
  .tracker-overview { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .aggregate-head { flex-direction: column; gap: 6px; }
  .tracker-row-head { flex-direction: column; }
  .tracker-targets { justify-content: flex-start; width: 100%; }
  .tracker-progress-grid { grid-template-columns: 1fr; gap: 8px; }
  .progress-block p { min-height: 0; }
  .eta-line { flex-direction: column; gap: 3px; }
  .tracker-mode-switch { width: 100%; display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); }
  .tracker-mode-switch button { min-height: 38px; }
  .growth-compact-list { grid-template-columns: 1fr; }
  .tracker-mode-summary { flex-wrap: wrap; }
  .tracker-mode-actions { width: 100%; justify-content: flex-end; }
  .tracker-edit-button { margin-left: 0; }
  .tracker-view-editor { align-items: stretch; }
  .tracker-view-search input[type="search"] { width: 100%; min-width: 0; flex: auto }
  .tracker-view-selected { align-items: stretch }
  .tracker-view-edit-row { align-items: stretch }
  .tracker-view-edit-head { min-width: 0 }
  .tracker-view-edit-row label { justify-content: space-between; width: 100% }
  .tracker-view-edit-row label select, .tracker-view-edit-row label input { flex: 1; width: auto; min-width: 0 }
  .tracker-view-row-actions { justify-content: stretch; width: 100% }
  .tracker-view-row-actions button { flex: 1 }
  .tracker-view-results { grid-template-columns: repeat(auto-fill, minmax(70px, 1fr)); max-height: 240px }
  .groups-manage-head { flex-direction: column; align-items: stretch }
  .group-create input { width: 100%; flex: 1 }
  .group-create { display: flex }
  .groups-grid { grid-template-columns: 1fr }
  .tracker-view-actions button { flex: 1 }
  .heart-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
}
@media (prefers-reduced-motion: reduce) { .spin { animation: none; } .progress-track i { transition: none; } }
</style>
