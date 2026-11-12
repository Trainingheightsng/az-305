# Lab: Implement Azure Monitor in the Azure Portal

In this lab you set up Azure Monitor for a virtual machine using only the Azure portal. You collect its performance data and the subscription's Activity log in one Log Analytics workspace, query the data, and get an email alert when CPU runs high.

## Contents

- [Lab overview](#lab-overview)
- [Exercise 1: Create a resource group and Log Analytics workspace](#exercise-1-create-a-resource-group-and-log-analytics-workspace)
- [Exercise 2: Create a test virtual machine](#exercise-2-create-a-test-virtual-machine)
- [Exercise 3: Enable VM insights with the Azure Monitor Agent](#exercise-3-enable-vm-insights-with-the-azure-monitor-agent)
- [Exercise 4: Send the Activity log to the workspace](#exercise-4-send-the-activity-log-to-the-workspace)
- [Exercise 5: Query logs with KQL](#exercise-5-query-logs-with-kql)
- [Exercise 6: Create an action group and a CPU alert](#exercise-6-create-an-action-group-and-a-cpu-alert)
- [Exercise 7: Pin a chart to a dashboard, then clean up](#exercise-7-pin-a-chart-to-a-dashboard-then-clean-up)
- [What you built](#what-you-built)
- [References](#references)

## Lab overview

**Duration:** about 75 minutes.

**You will:**

1. Create a Log Analytics workspace to store logs.
2. Create a small test VM.
3. Turn on VM insights, which installs the Azure Monitor Agent.
4. Send the Activity log to the workspace.
5. Query the data with Kusto Query Language (KQL).
6. Create an action group and a CPU alert.
7. Pin a chart to a dashboard, then delete everything.

**Before you start:**

- An Azure subscription where you have the **Contributor** or **Owner** role on the **whole subscription**. A role on a resource group alone is not enough, because Exercise 4 changes a subscription-level setting.
- A web browser signed in to the [Azure portal](https://portal.azure.com).
- An email address you can check during the lab.
- Enough quota for one small VM (2 vCPUs) in your region.

**Names used in this lab.** Replace `<initials>` with your own initials so names don't clash with other participants.

| Item | Value |
| --- | --- |
| Region | South Africa North |
| Resource group | `rg-monitor-lab-<initials>` |
| Log Analytics workspace | `law-monitor-lab-<initials>` |
| Virtual machine | `vm-monitor-<initials>` |
| Action group | `ag-monitor-lab` |

> [!NOTE]
> If South Africa North isn't available in your subscription, use any other region, and use the same region for every resource.

## Exercise 1: Create a resource group and Log Analytics workspace

A Log Analytics workspace is where Azure Monitor stores log data so you can query it.

1. In the Azure portal search bar, type **Resource groups** and select it.
2. Select **+ Create**. Choose your subscription, enter `rg-monitor-lab-<initials>`, select your region, then select **Review + create** and **Create**.
3. In the search bar, type **Log Analytics workspaces** and select it.
4. Select **+ Create** and fill in:
   - **Resource group:** `rg-monitor-lab-<initials>`
   - **Name:** `law-monitor-lab-<initials>`
   - **Region:** your lab region
5. Select **Review + create**, then **Create**. Wait for the deployment to finish, then select **Go to resource**.

**Check:** the workspace **Overview** page opens and shows the workspace name and region.

## Exercise 2: Create a test virtual machine

1. In the search bar, type **Virtual machines** and select it. Select **+ Create** > **Azure virtual machine**.
2. On the **Basics** tab, fill in:
   - **Resource group:** `rg-monitor-lab-<initials>`
   - **Virtual machine name:** `vm-monitor-<initials>`
   - **Region:** your lab region
   - **Availability options:** No infrastructure redundancy required
   - **Image:** any **Windows Server 2022 Datacenter** image (the default x64 Gen2 image is fine)
   - **Size:** `Standard_B2s`. If it shows as unavailable, select **See all sizes** and pick `Standard_D2s_v5`.
   - **Username** and **Password:** choose your own and note them down
   - **Public inbound ports:** None
3. Select the **Monitoring** tab. If it offers to enable alerts or enhanced monitoring, leave those **off**. You set up monitoring yourself in Exercises 3 and 6.
4. Select **Review + create**, then **Create**. Deployment takes about 3 to 5 minutes.

**Check:** open the VM. Its **Status** shows **Running**.

> [!NOTE]
> We don't open RDP to the internet. You don't need to sign in to the VM for this lab.

## Exercise 3: Enable VM insights with the Azure Monitor Agent

Enabling monitoring installs the Azure Monitor Agent on the VM and creates a data collection rule (DCR). The DCR tells the agent what to collect and which workspace to send it to. In this lab we use the **logs-based** option, so the data goes to your Log Analytics workspace and you can query it in Exercise 5.

The VM must be **Running** before you start.

1. Open your VM. In its menu, select **Monitor**. You see charts with no data and a message offering to enable enhanced monitoring.
2. Select **Configure**. The **Configure monitor** page opens.
3. Under metrics, select the **logs-based** metrics option. If you can clear **OpenTelemetry metrics**, clear it to keep the lab simple.
4. Select **Customize infrastructure monitoring**. Under **Workspaces**, for the Log Analytics workspace, select `law-monitor-lab-<initials>`.
5. Leave **Data collection rule** on its default, which creates a new rule.
6. Clear **Enable recommended alerts**. You create your own alert in Exercise 6.
7. Select **Review + Enable**, then **Enable**.

The agent installs in a few minutes, and data appears a few minutes after that. Carry on with Exercise 4 while you wait.

**Check:** later, return to **Monitor** on the VM. The CPU, memory and disk charts now show data.

> [!TIP]
> **If your portal looks different.** Microsoft is rolling out this new experience gradually. If your VM shows **Insights** under **Monitoring** instead, select **Insights** > **Enable**, keep **Azure Monitor agent**, select **Create new** under **Data collection rule**, choose `law-monitor-lab-<initials>` as the workspace, then select **Create** and **Configure**.

## Exercise 4: Send the Activity log to the workspace

The Activity log records who changed what in the subscription, such as creating or stopping a VM. Azure keeps it for 90 days. Sending it to the workspace lets you query it alongside your other data and keep it for longer.

1. In the search bar, type **Monitor** and select it.
2. In the Monitor menu, select **Activity log**, then select **Export Activity Logs**.
3. Select your subscription, then select **+ Add diagnostic setting**.
4. Fill in:
   - **Diagnostic setting name:** `send-activity-to-law`
   - **Categories:** tick **Administrative**, **Security**, **Policy** and **Alert**
   - **Destination details:** tick **Send to Log Analytics workspace** and select `law-monitor-lab-<initials>`
5. Select **Save**.
6. To create some activity, go to your VM, select **Stop**, wait until it shows **Stopped**, then select **Start**.

**Check:** the new setting appears in the list of diagnostic settings. Events can take up to 15 minutes to reach the workspace.

> [!NOTE]
> If your subscription already has a diagnostic setting for the Activity log, add a second one. Ask your trainer if you can't save it.

## Exercise 5: Query logs with KQL

1. Open your workspace `law-monitor-lab-<initials>` and select **Logs**. Close the **Queries** window if it opens.
2. If the editor shows **Simple mode**, switch to **KQL mode**.
3. Paste each query below, select **Run**, and look at the results.

**Query 1: is the VM reporting in?** This lists each machine and when it last sent a heartbeat.

```kusto
Heartbeat
| summarize LastSeen = max(TimeGenerated) by Computer
```

**Query 2: CPU over time.** This draws a chart of average CPU every 5 minutes.

```kusto
InsightsMetrics
| where Namespace == "Processor" and Name == "UtilizationPercentage"
| summarize AvgCPU = avg(Val) by bin(TimeGenerated, 5m), Computer
| render timechart
```

**Query 3: who stopped or started the VM?** This reads the Activity log you exported in Exercise 4.

```kusto
AzureActivity
| where OperationNameValue has "virtualMachines"
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue
| order by TimeGenerated desc
```

**Check:** query 1 shows your VM, query 2 shows a line chart, and query 3 shows your stop and start actions with your account as the **Caller**.

> [!TIP]
> No results yet? Data can take 10 to 15 minutes to arrive. Move on to Exercise 6 and come back.

## Exercise 6: Create an action group and a CPU alert

An alert rule watches a signal, such as CPU. When the condition is met, it calls an action group, which decides who to notify and how.

1. Open your VM. In its menu, under **Monitoring**, select **Alerts**, then **+ Create** > **Alert rule**.
2. On the **Condition** tab:
   - **Signal name:** Percentage CPU
   - **Threshold:** Static
   - **Aggregation type:** Average
   - **Operator:** Greater than
   - **Threshold value:** `80`
   - **Check every:** 1 minute. **Lookback period:** 5 minutes.
3. On the **Actions** tab, select **Create action group** and fill in:
   - **Resource group:** `rg-monitor-lab-<initials>`
   - **Action group name:** `ag-monitor-lab`
   - **Display name:** `MonitorLab`
   - **Notifications** tab: **Notification type:** Email/SMS message/Push/Voice. Tick **Email**, enter your email address, and name the notification `email-me`.
4. Select **Review + create**, then **Create**. You are taken back to the alert rule.
5. On the **Details** tab:
   - **Severity:** 2 - Warning
   - **Alert rule name:** `alert-high-cpu`
6. Select **Review + create**, then **Create**.

**Check:** you get an email saying you were added to the action group.

**See the alert fire.** The test VM is almost idle, so it won't reach 80% CPU. Open **Alerts** > **Alert rules**, select `alert-high-cpu`, and select **Edit**. On the **Condition** tab, change **Operator** to **Less than**, keep the threshold at `80`, and save. An idle VM is always below 80%, so within about 5 to 10 minutes the alert shows as **Fired** under **Alerts**, and you get an email. Then edit the rule again and set **Operator** back to **Greater than**.

## Exercise 7: Pin a chart to a dashboard, then clean up

### Pin a chart

1. Open your VM. Under **Monitoring**, select **Metrics**.
2. Set **Metric** to **Percentage CPU** and **Aggregation** to **Avg**.
3. Select **Save to dashboard** > **Pin to dashboard**. Choose **Create new**, name it `Monitor lab`, and select **Create and pin**.
4. In the portal menu, select **Dashboard** and switch to `Monitor lab`.

**Check:** the CPU chart appears on the dashboard.

### Clean up

Delete everything so it stops costing money.

1. Open **Monitor** > **Activity log** > **Export Activity Logs**. Select your subscription, select **Edit setting** next to `send-activity-to-law`, then select **Delete**. This setting belongs to the subscription, so deleting the resource group won't remove it.
2. Open **Resource groups**, select `rg-monitor-lab-<initials>`, then select **Delete resource group**. Type the name to confirm and select **Delete**.
3. Open **Dashboard**, switch to `Monitor lab`, and select **Delete**.
4. In the search bar, type **Azure Monitor workspaces**. If one was created for your VM in Exercise 3, delete it. Do the same for **Data collection rules**, and for any resource group named **DefaultResourceGroup-** followed by your region code, if you see one and nobody else uses it.

> [!NOTE]
> You may also see a resource group called **NetworkWatcherRG**. Azure creates it automatically when you create a virtual network. It costs nothing, so leave it unless your trainer says otherwise.

## What you built

| Azure Monitor feature | What it did in this lab |
| --- | --- |
| Log Analytics workspace | Stored the VM's performance data and the Activity log in one place |
| Azure Monitor Agent and data collection rule | Collected performance data from inside the VM |
| VM insights | Showed ready-made performance charts for the VM |
| Diagnostic setting | Sent the subscription's Activity log to the workspace |
| KQL | Queried heartbeats, CPU and Activity log entries |
| Alert rule and action group | Emailed you when CPU crossed a threshold |
| Dashboard | Kept a CPU chart in one place for the team |

## References

- [Tutorial: Enable enhanced monitoring for an Azure virtual machine](https://learn.microsoft.com/en-us/azure/azure-monitor/vm/tutorial-enable-monitoring)
- [Enable VM insights in the Azure portal](https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-enable-portal)
- [Send Azure Monitor activity log data](https://learn.microsoft.com/azure/azure-monitor/platform/activity-log)
- [Create diagnostic settings in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/create-diagnostic-settings)