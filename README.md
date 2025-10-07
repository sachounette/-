function testSearchTicket() {
  const testPerson = 'user@example.com'; // замените на email или имя пользователя
  const jiraDate = '2025-10-07'; // конкретная дата для теста
  const reporterJql = `reporter ~ "${escapeJql(testPerson)}"`;
  const queueJql = `queue = ${CONFIG.QUEUE_ID}`;
  const dateJql = `created >= "${jiraDate} 00:00" AND created <= "${jiraDate} 23:59"`;

  const jql = `project = ${CONFIG.PROJECT_KEY} AND ${reporterJql} AND ${queueJql} AND (${dateJql}) ORDER BY created DESC`;
  Logger.log('JQL для теста: ' + jql);

  // Используем ваши переменные из CONFIG
  const base = CONFIG.JIRA_BASE_URL.replace(/\/$/, '');
  const url = `${base}/rest/api/2/search?jql=${encodeURIComponent(jql)}&fields=key,summary,reporter,created&maxResults=${CONFIG.MAX_RESULTS}`;
  const auth = Utilities.base64Encode(CONFIG.JIRA_USERNAME + ':' + CONFIG.JIRA_API_TOKEN);
  const options = {
    method: 'get',
    headers: { 'Authorization': 'Basic ' + auth, 'Accept': 'application/json' },
    muteHttpExceptions: true
  };

  try {
    const resp = UrlFetchApp.fetch(url, options);
    const code = resp.getResponseCode();
    if (code < 200 || code >= 300) {
      throw new Error(`Jira API returned ${code}: ${resp.getContentText().slice(0,300)}`);
    }

    const data = JSON.parse(resp.getContentText());
    const issues = data.issues || [];

    if (issues.length) {
      issues.forEach(issue => {
        Logger.log(`Найден тикет: ${issue.key} — ${issue.fields.summary}`);
      });
    } else {
      Logger.log('Тикетов не найдено.');
    }
  } catch (err) {
    Logger.log('Ошибка поиска: ' + err.message);
  }
}
