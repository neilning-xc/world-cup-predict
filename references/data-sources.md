# 数据源URL和解析规则

**使用 chrome-devtools-mcp 工具获取所有数据。**

## 1. 当天比赛列表

### ESPN (推荐，包含赔率)
```
URL: https://www.espn.com/soccer/schedule/_/date/{YYYYMMDD}/league/fifa.world

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["MATCH"] - 等待页面加载
3. take_snapshot - 获取页面快照
4. evaluate_script - 提取结构化数据
```

**JavaScript 提取脚本:**
```javascript
() => {
  const matches = [];
  document.querySelectorAll('[data-testid="schedContainer"]').forEach(container => {
    const dateHeader = container.querySelector('h3')?.textContent || '';
    container.querySelectorAll('[data-testid="gameCard"]').forEach(card => {
      const teams = card.querySelectorAll('[data-testid="teamName"]');
      const time = card.querySelector('[data-testid="gameTime"]')?.textContent || '';
      const odds = card.querySelector('[data-testid="odds"]')?.textContent || '';
      matches.push({
        date: dateHeader,
        home: teams[0]?.textContent || '',
        away: teams[1]?.textContent || '',
        time: time,
        odds: odds
      });
    });
  });
  return matches;
}
```

### FIFA官网 (备用)
```
URL: https://www.fifa.com/fifaplus/en/tournaments/mens/worldcup/canadamexicousa2026/schedule

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Schedule"]
3. take_snapshot
```

## 2. 博彩赔率 (F1: 25%)

### ESPN (从比赛列表获取)
```
已在比赛列表中获取，无需单独请求
字段: 让球盘、大小盘
```

### OddsPortal (详细赔率)
```
URL: https://www.oddsportal.com/football/world/world-cup-2026/{team1}-{team2}/

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Odds"]
3. take_snapshot
4. evaluate_script 提取赔率
```

**JavaScript 提取脚本:**
```javascript
() => {
  const odds = {};
  document.querySelectorAll('.odds-tab').forEach(tab => {
    const type = tab.querySelector('.name')?.textContent || '';
    const values = [];
    tab.querySelectorAll('.odds-value').forEach(val => {
      values.push(parseFloat(val.textContent));
    });
    odds[type] = values;
  });
  return odds;
}
```

### 转换公式
```
主胜概率 = (1 / 主胜赔率) / (1/主胜 + 1/平局 + 1/客胜)
平局概率 = (1 / 平局赔率) / (1/主胜 + 1/平局 + 1/客胜)
客胜概率 = (1 / 客胜赔率) / (1/主胜 + 1/平局 + 1/客胜)
```

### 评分标准
| 赔率差值 | 评分 |
|----------|------|
| 主队赔率 < 1.5 | 主队 90分 |
| 主队赔率 1.5-2.0 | 主队 70分 |
| 主队赔率 2.0-2.5 | 主队 55分 |
| 赔率接近 (差<0.3) | 双方 50分 |

## 3. FIFA排名 (F2: 15%)

### FIFA官网
```
URL: https://www.fifa.com/fifa-world-ranking/men

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Rank"]
3. take_snapshot
4. evaluate_script 提取排名数据
```

**JavaScript 提取脚本:**
```javascript
() => {
  const rankings = [];
  document.querySelectorAll('table tbody tr').forEach(row => {
    const cells = row.querySelectorAll('td');
    rankings.push({
      rank: parseInt(cells[0]?.textContent) || 0,
      team: cells[1]?.textContent?.trim() || '',
      points: parseInt(cells[3]?.textContent) || 0
    });
  });
  return rankings;
}
```

### 评分标准
| 排名差 | 评分差 |
|--------|--------|
| 差 > 30 | 高排名队 +30分 |
| 差 20-30 | 高排名队 +20分 |
| 差 10-20 | 高排名队 +10分 |
| 差 < 10 | 双方 +0分 |

## 4. 近期战绩 (F3: 15%)

### Transfermarkt
```
URL: https://www.transfermarkt.com/{team}/spielplandatum/verein/{id}

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Result"]
3. take_snapshot
4. evaluate_script 提取比赛结果
```

**JavaScript 提取脚本:**
```javascript
() => {
  const results = [];
  document.querySelectorAll('.responsive-table .items tbody tr').forEach(row => {
    const date = row.querySelector('.date')?.textContent?.trim() || '';
    const opponent = row.querySelector('.opponent a')?.textContent?.trim() || '';
    const score = row.querySelector('.result')?.textContent?.trim() || '';
    const competition = row.querySelector('.wettbewerb')?.textContent?.trim() || '';
    results.push({ date, opponent, score, competition });
  });
  return results.slice(0, 10);
}
```

### ESPN (备用)
```
URL: https://www.espn.com/soccer/team/results/_/id/{team_id}/league/FIFA.WORLD

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Result"]
3. take_snapshot
```

### 评分公式
```
状态分 = (胜×3 + 平×1) / 30 × 100
进球效率 = 总进球 / 场次
失球率 = 总失球 / 场次
综合分 = 状态分 × 0.6 + 进球效率×10 × 0.2 + (10-失球率×10) × 0.2
```

## 5. 历史交锋 (F4: 10%)

### Transfermarkt Head-to-Head
```
URL: https://www.transfermarkt.com/{team1}_head2head/verein/{id1}/gegner_id/{id2}

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Head"]
3. take_snapshot
4. evaluate_script 提取交锋记录
```

**JavaScript 提取脚本:**
```javascript
() => {
  const h2h = [];
  document.querySelectorAll('.box .content table tbody tr').forEach(row => {
    const date = row.querySelector('td:first-child')?.textContent?.trim() || '';
    const homeTeam = row.querySelector('td:nth-child(2)')?.textContent?.trim() || '';
    const score = row.querySelector('td:nth-child(3)')?.textContent?.trim() || '';
    const awayTeam = row.querySelector('td:nth-child(4)')?.textContent?.trim() || '';
    h2h.push({ date, homeTeam, score, awayTeam });
  });
  return h2h.slice(0, 10);
}
```

### 评分标准
| 近5场交锋 | 评分 |
|-----------|------|
| 4胜以上 | 80分 |
| 3胜 | 65分 |
| 2胜1平2负 | 50分 (均衡) |
| 1胜以下 | 35分 |

## 6. 阵容信息 (F6: 10%)

### Transfermarkt 阵容页面
```
URL: https://www.transfermarkt.com/{team}/kader/verein/{id}

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Player"]
3. take_snapshot
4. evaluate_script 提取球员列表
```

**JavaScript 提取脚本:**
```javascript
() => {
  const players = [];
  document.querySelectorAll('.items tbody tr').forEach(row => {
    const name = row.querySelector('.hauptlink a')?.textContent?.trim() || '';
    const position = row.querySelector('.inline-table td:last-child')?.textContent?.trim() || '';
    const injury = row.querySelector('.verletzt')?.textContent?.trim() || '';
    const suspended = row.querySelector('.gelbgesperrt')?.textContent?.trim() || '';
    players.push({ name, position, injury, suspended });
  });
  return players;
}
```

### 伤病影响评分
| 伤缺球员级别 | 扣分 |
|--------------|------|
| 核心球星 | -20分 |
| 主力球员 | -10分 |
| 轮换球员 | -3分 |

## 7. 预选赛表现 (F5: 10%)

### FIFA预选赛页面
```
URL: https://www.fifa.com/fifaplus/en/tournaments/mens/worldcup/qualifiers

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Standing"]
3. take_snapshot
4. evaluate_script 提取积分榜
```

**JavaScript 提取脚本:**
```javascript
() => {
  const standings = [];
  document.querySelectorAll('table tbody tr').forEach(row => {
    const cells = row.querySelectorAll('td');
    standings.push({
      rank: parseInt(cells[0]?.textContent) || 0,
      team: cells[1]?.textContent?.trim() || '',
      played: parseInt(cells[2]?.textContent) || 0,
      wins: parseInt(cells[3]?.textContent) || 0,
      draws: parseInt(cells[4]?.textContent) || 0,
      losses: parseInt(cells[5]?.textContent) || 0,
      gf: parseInt(cells[6]?.textContent) || 0,
      ga: parseInt(cells[7]?.textContent) || 0,
      points: parseInt(cells[9]?.textContent) || 0
    });
  });
  return standings;
}
```

### 评分标准
| 预选赛排名 | 评分 |
|------------|------|
| 小组第1 | 85分 |
| 小组第2 | 70分 |
| 小组第3 | 55分 |
| 附加赛晋级 | 60分 |

## 8. 世界杯小组赛积分榜 (新增)

### ESPN积分榜
```
URL: https://www.espn.com/soccer/standings/_/league/FIFA.WORLD

获取方式:
1. navigate_page: 导航到URL
2. wait_for: ["Group"]
3. take_snapshot
4. evaluate_script 提取各组积分榜
```

**JavaScript 提取脚本:**
```javascript
() => {
  const groups = [];
  document.querySelectorAll('.Team').forEach(table => {
    const groupName = table.closest('.Table')?.querySelector('h3')?.textContent || '';
    const teams = [];
    table.querySelectorAll('tbody tr').forEach(row => {
      const cells = row.querySelectorAll('td');
      teams.push({
        rank: parseInt(cells[0]?.textContent) || 0,
        team: cells[1]?.textContent?.trim() || '',
        gp: parseInt(cells[2]?.textContent) || 0,
        w: parseInt(cells[3]?.textContent) || 0,
        d: parseInt(cells[4]?.textContent) || 0,
        l: parseInt(cells[5]?.textContent) || 0,
        gf: parseInt(cells[6]?.textContent) || 0,
        ga: parseInt(cells[7]?.textContent) || 0,
        gd: parseInt(cells[8]?.textContent) || 0,
        pts: parseInt(cells[9]?.textContent) || 0
      });
    });
    groups.push({ group: groupName, teams });
  });
  return groups;
}
```

## 9. 主客场因素 (F7: 8%)

### 评分标准
| 情况 | 评分差 |
|------|--------|
| 本大洲球队 vs 其他大洲 | +15分 |
| 邻国球队 | +10分 |
| 同一大洲 | +5分 |
| 跨大洲 | +0分 |

## 10. 赛程/体能 (F8: 4%)

### 评分标准
| 休息天数 | 评分 |
|----------|------|
| ≥4天 | 80分 |
| 3天 | 65分 |
| 2天 | 50分 |
| <2天 | 35分 |

## 11. 战术风格克制 (F9: 3%)

### 评分标准
| 风格关系 | 评分差 |
|----------|--------|
| 明显克制 | +10分 |
| 轻微克制 | +5分 |
| 无明显关系 | +0分 |

### 常见克制关系
- 高位压迫 克制 控球型
- 防守反击 克制 高位压迫
- 两翼齐飞 克制 中路密集防守
- 控球型 克制 防守反击
