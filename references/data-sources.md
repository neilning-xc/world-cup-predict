# 数据源URL和解析规则

**综合使用 chrome-devtools-mcp 和 WebSearch 工具获取数据。**
由于部分体育网站（如 Transfermarkt, OddsPortal, FIFA官网）部署了强力的 Cloudflare 反爬虫措施，我们改用更可靠的数据源（如 ESPN, 维基百科）以及 WebSearch 搜索。

## 1. 当天比赛列表与赔率 (F1: 25%)

### ESPN (推荐，稳定无强反爬，包含赔率)
```
URL: https://www.espn.com/soccer/schedule/_/date/{YYYYMMDD}/league/fifa.world

获取方式:
1. chrome-devtools-mcp -> navigate_page: 导航到URL
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
      const matchLink = card.querySelector('a.AnchorLink')?.href || '';
      matches.push({
        date: dateHeader,
        home: teams[0]?.textContent || '',
        away: teams[1]?.textContent || '',
        time: time,
        odds: odds,
        link: matchLink
      });
    });
  });
  return matches;
}
```
*注：ESPN的比赛列表通常会直接显示Moneyline赔率（如 +150, -120 等）。如果列表未显示，可通过 WebSearch 搜索 "{主队} vs {客队} odds world cup 2026" 获取。*

## 2. FIFA排名 (F2: 15%)

### 维基百科 (静态页面，100%无反爬)
```
URL: https://en.wikipedia.org/wiki/FIFA_Men%27s_World_Ranking

获取方式:
1. chrome-devtools-mcp -> navigate_page: 导航到URL
2. take_snapshot
3. evaluate_script 提取排名数据
```

**JavaScript 提取脚本:**
```javascript
() => {
  const rankings = [];
  // 维基百科的排名表格通常是第一个或第二个带有 'wikitable' 类的表格
  document.querySelectorAll('table.wikitable tbody tr').forEach(row => {
    const cells = row.querySelectorAll('td, th');
    if (cells.length >= 3) {
      const rank = parseInt(cells[0]?.textContent?.trim()) || 0;
      const team = cells[1]?.textContent?.trim() || cells[2]?.textContent?.trim() || '';
      if (rank > 0 && team) {
        rankings.push({ rank, team });
      }
    }
  });
  return rankings;
}
```

## 3. 近期战绩 (F3: 15%)

由于 Transfermarkt 有强反爬，改用 **WebSearch 工具** 或 **ESPN球队主页**。

### 方案A: WebSearch (推荐)
使用 `WebSearch` 工具搜索：`"{team name} national football team recent match results 2026"`
直接从搜索结果摘要中提取该队最近5-10场的胜平负情况。

### 方案B: ESPN球队赛程页
```
URL: https://www.espn.com/soccer/team/results/_/name/{team_abbreviation} 
(如 /name/bra 巴西, /name/ger 德国)

获取方式:
1. navigate_page
2. take_snapshot 提取比分
```

## 4. 历史交锋 (F4: 5%)

### 方案A: WebSearch (推荐)
使用 `WebSearch` 工具搜索：`"{team1} vs {team2} football head to head stats history"`
从搜索结果中提取两队历史交锋的胜平负记录。

### 方案B: 11v11 (备用，反爬较弱)
```
URL: https://www.11v11.com/teams/{team1}/tab/opposingTeams/opposition/{team2}/

获取方式:
1. chrome-devtools-mcp -> navigate_page
2. take_snapshot 提取交锋记录
```

## 5. 阵容信息与伤病 (F6: 15%)

由于 Transfermarkt 有强反爬，改用 **WebSearch 工具**。

### WebSearch (推荐)
使用 `WebSearch` 工具搜索：`"{team name} world cup 2026 squad injuries suspensions news"`
从搜索结果中提取是否有核心球员伤缺或停赛。

## 6. 预选赛表现 / 小组赛积分 (F5: 10%)

### 维基百科或ESPN
对于预选赛，使用维基百科：
```
URL: https://en.wikipedia.org/wiki/2026_FIFA_World_Cup_qualification
```

对于正赛小组赛积分榜，使用 ESPN：
```
URL: https://www.espn.com/soccer/standings/_/league/FIFA.WORLD

获取方式:
1. navigate_page
2. wait_for: ["Group"]
3. take_snapshot
4. evaluate_script 提取各组积分榜
```

**JavaScript 提取脚本:**
```javascript
() => {
  const groups = [];
  document.querySelectorAll('.Table__Title').forEach(title => {
    const groupName = title.textContent || '';
    const table = title.nextElementSibling;
    const teams = [];
    if (table) {
      table.querySelectorAll('tbody tr').forEach(row => {
        const teamName = row.querySelector('.hide-mobile')?.textContent?.trim() || '';
        const stats = row.querySelectorAll('.stat-cell');
        if (teamName && stats.length >= 8) {
          teams.push({
            team: teamName,
            gp: parseInt(stats[0]?.textContent) || 0,
            w: parseInt(stats[1]?.textContent) || 0,
            d: parseInt(stats[2]?.textContent) || 0,
            l: parseInt(stats[3]?.textContent) || 0,
            gf: parseInt(stats[4]?.textContent) || 0,
            ga: parseInt(stats[5]?.textContent) || 0,
            gd: parseInt(stats[6]?.textContent) || 0,
            pts: parseInt(stats[7]?.textContent) || 0
          });
        }
      });
    }
    groups.push({ group: groupName, teams });
  });
  return groups;
}
```

## 7. 主客场因素 (F7: 8%)
无需联网，直接根据球队所在大洲判断（2026世界杯在美加墨举办，中北美球队有一定主场优势）。

## 8. 赛程/体能 (F8: 4%)
根据 ESPN 赛程表中的比赛间隔天数计算。

## 9. 战术风格克制 (F9: 3%)
基于大语言模型的内在知识库进行判断。
