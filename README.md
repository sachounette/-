const CONFIG = {
  SOURCE_SHEET: 'UAR Report',
  HISTORY_SHEET: 'UAR_History',
  PROJECT_KEY: 'FNCRM',
  MAX_RESULTS: 5,
  CREATE_IF_NOT_FOUND: true,
  DRY_RUN: true,
  DEFAULT_ISSUE_TYPE: 'Task',
  DEFAULT_SUMMARY_TEMPLATE: 'UAR: {person} absent {date}',
  DEFAULT_DESCRIPTION_TEMPLATE: 'Автоматически создано скриптом UAR Report.\nСотрудник: {person}\nДата отсутствия: {date}\nТимлид: {teamLead}\nКоманда: {team}\n',

  JIRA_BASE_URL: PropertiesService.getScriptProperties().getProperty('JIRA_BASE_URL') || 'https://your-jira.example.com',
  JIRA_USERNAME: PropertiesService.getScriptProperties().getProperty('JIRA_USERNAME') || 'your.username',
  JIRA_API_TOKEN: PropertiesService.getScriptProperties().getProperty('JIRA_API_TOKEN') || 'your_api_token',

  SLACK_API_TOKEN: PropertiesService.getScriptProperties().getProperty('SLACK_API_TOKEN') || 'xoxb-your-slack-token',
  SLACK_CHANNEL: '#ua-report'
};

function dailyJiraCheck(){
  const reportRows = checkJiraTicketsForUAR();
  sendSlackReport(reportRows);
}

function checkJiraTicketsForUAR(){
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = ss.getSheetByName(CONFIG.SOURCE_SHEET);
  if(!sheet) throw new Error('Sheet "' + CONFIG.SOURCE_SHEET + '" not found');
  
  const historySheet = ss.getSheetByName(CONFIG.HISTORY_SHEET) || ss.insertSheet(CONFIG.HISTORY_SHEET);

  const lastRow = sheet.getLastRow();
  if(lastRow < 2) return [];

  const readCols = Math.max(4, sheet.getLastColumn());
  const rows = sheet.getRange(2, 1, lastRow - 1, readCols).getValues();

  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  const yDateStr = formatDateForJira(yesterday);

  const output = [];
  const slackReportRows = [];

  for(let i = 0; i < rows.length; i++){
    const r = rows[i];
    const dateCell = r[0];
    const person = String(r[1] || '').trim();
    const teamLead = String(r[2] || '').trim();
    const team = String(r[3] || '').trim();

    if(!dateCell || !person){
      output.push([...r, '', '', '']);
      continue;
    }

    let jiraDate;
    try{ jiraDate = formatDateForJira(dateCell); } catch(e){
      output.push([...r, '', '', 'ERROR: invalid date']);
      continue;
    }

    if(jiraDate !== yDateStr){
      output.push([...r, '', '', '']);
      continue;
    }

    slackReportRows.push({email: person, team: team});

    const dateJql = `created >= "${jiraDate} 00:00" AND created <= "${jiraDate} 23:59"`;
    const reporterJql = `reporter ~ "${escapeJql(person)}"`;

    let jql = `project = ${CONFIG.PROJECT_KEY} AND ${reporterJql} AND (${dateJql}) ORDER BY created DESC`;

    try{
      const issues = searchJira(jql, CONFIG.MAX_RESULTS);
      if(issues && issues.length){
        const issueLink = `${CONFIG.JIRA_BASE_URL.replace(/\/$/, '')}/browse/${issues[0].key}`;
        output.push([...r, issueLink, '', 'FOUND']);
      } else if(CONFIG.CREATE_IF_NOT_FOUND){
        const summary = CONFIG.DEFAULT_SUMMARY_TEMPLATE.replace('{person}', person).replace('{date}', jiraDate).replace('{teamLead}', teamLead);
        const description = CONFIG.DEFAULT_DESCRIPTION_TEMPLATE.replace('{person}', person).replace('{date}', jiraDate).replace('{teamLead}', teamLead).replace('{team}', team);

        if(CONFIG.DRY_RUN){
          output.push([...r, '', 'DRY_RUN: would create issue with summary: ' + summary, '']);
        } else {
          try{
            const newKey = createJiraIssue(summary, description, CONFIG.DEFAULT_ISSUE_TYPE);
            const issueLink = `${CONFIG.JIRA_BASE_URL.replace(/\/$/, '')}/browse/${newKey}`;
            output.push([...r, '', issueLink, 'CREATED']);
          } catch(creErr){
            output.push([...r, '', '', 'ERROR_CREATE: ' + creErr.message]);
          }
        }
      } else {
        output.push([...r, '', '', '']);
      }
    } catch(err){
      output.push([...r, '', '', 'ERROR: ' + err.message]);
    }
  }

  const startRow = historySheet.getLastRow() + 1;
  historySheet.getRange(startRow, 1, output.length, output[0].length).setValues(output);

  return slackReportRows;
}

function sendSlackReport(rows){
  if(!rows.length) return;
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  const dateStr = Utilities.formatDate(yesterday, Session.getScriptTimeZone(), 'yyyy-MM-dd');

  let text = `*UAR Report for ${dateStr}:*\n`;

  rows.forEach(r => {
    let slackTag = r.email; // default: email
    try {
      const user = getSlackUserIdByEmail(r.email);
      if(user) slackTag = `<@${user.id}>`;
    } catch(e){}
    text += `• ${slackTag} (${r.team})\n`;
  });

  const url = 'https://slack.com/api/chat.postMessage';
  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: { 'Authorization': 'Bearer ' + CONFIG.SLACK_API_TOKEN },
    payload: JSON.stringify({ channel: CONFIG.SLACK_CHANNEL, text: text })
  };

  UrlFetchApp.fetch(url, options);
}

function getSlackUserIdByEmail(email){
  const url = 'https://slack.com/api/users.lookupByEmail?email=' + encodeURIComponent(email);
  const options = { method: 'get', headers: { 'Authorization': 'Bearer ' + CONFIG.SLACK_API_TOKEN } };
  const resp = UrlFetchApp.fetch(url, options);
  const data = JSON.parse(resp.getContentText());
  return data.ok ? data.user : null;
}

function formatDateForJira(dateCell){
  let d = dateCell;
  if(!(d instanceof Date)){ d = new Date(String(dateCell)); if(isNaN(d)) throw new Error('Invalid date: ' + dateCell); }
  const yyyy = d.getFullYear();
  const mm = ('0' + (d.getMonth() + 1)).slice(-2);
  const dd = ('0' + d.getDate()).slice(-2);
  return `${yyyy}-${mm}-${dd}`;
}

function escapeJql(s){ return s.replace(/"/g, '\\"'); }

function searchJira(jql, maxResults){
  const base = CONFIG.JIRA_BASE_URL.replace(/\/$/, '');
  const url = `${base}/rest/api/2/search?jql=${encodeURIComponent(jql)}&fields=key,summary,reporter,created&maxResults=${maxResults}`;
  const auth = Utilities.base64Encode(CONFIG.JIRA_USERNAME + ':' + CONFIG.JIRA_API_TOKEN);
  const options = { method: 'get', headers: {'Authorization': 'Basic ' + auth,'Accept': 'application/json'}, muteHttpExceptions: true };
  const resp = UrlFetchApp.fetch(url, options);
  const code = resp.getResponseCode();
  if(code < 200 || code >= 300){ let body = resp.getContentText(); try{ body = JSON.parse(body); }catch(e){} throw new Error(`Jira API returned ${code}: ${JSON.stringify(body).slice(0,300)}`); }
  const data = JSON.parse(resp.getContentText());
  return data.issues || [];
}

function createJiraIssue(summary, description, issueType){
  const base = CONFIG.JIRA_BASE_URL.replace(/\/$/, '');
  const url = `${base}/rest/api/2/issue`;
  const payload = { fields: { project: { key: CONFIG.PROJECT_KEY }, summary: summary, description: description, issuetype: { name: issueType } } };
  const auth = Utilities.base64Encode(CONFIG.JIRA_USERNAME + ':' + CONFIG.JIRA_API_TOKEN);
  const options = { method: 'post', contentType: 'application/json', payload: JSON.stringify(payload), headers: { 'Authorization': 'Basic ' + auth,'Accept': 'application/json' }, muteHttpExceptions: true };
  const resp = UrlFetchApp.fetch(url, options);
  const code = resp.getResponseCode();
  const bodyText = resp.getContentText();
  if(code < 200 || code >= 300){ let body = bodyText; try{ body = JSON.parse(bodyText); }catch(e){} throw new Error(`Create issue failed ${code}: ${JSON.stringify(body).slice(0,300)}`); }
  const data = JSON.parse(bodyText);
  return data.key;
}

function createDaily13Trigger(){
  const triggers = ScriptApp.getProjectTriggers();
  triggers.forEach(t => { if(t.getHandlerFunction() === 'dailyJiraCheck') ScriptApp.deleteTrigger(t); });
  ScriptApp.newTrigger('dailyJiraCheck').timeBased().everyDays(1).atHour(13).create();
}
