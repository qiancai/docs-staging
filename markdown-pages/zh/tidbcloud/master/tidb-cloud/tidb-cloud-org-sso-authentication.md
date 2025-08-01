---
title: 组织 SSO 认证
summary: 了解如何通过自定义组织认证登录 TiDB Cloud 控制台。
---

# 组织 SSO 认证

单点登录（SSO）是一种认证方案，使你的 TiDB Cloud [组织](/tidb-cloud/tidb-cloud-glossary.md#organization)中的成员能够使用身份提供商（IdP）的身份而不是电子邮件地址和密码登录 TiDB Cloud。

TiDB Cloud 支持以下两种 SSO 认证：

- [标准 SSO](/tidb-cloud/tidb-cloud-sso-authentication.md)：成员可以使用 GitHub、Google 或 Microsoft 认证方法登录 [TiDB Cloud 控制台](https://tidbcloud.com/)。标准 SSO 默认为 TiDB Cloud 中的所有组织启用。

- 云组织 SSO：成员可以使用组织指定的认证方法登录 TiDB Cloud 的自定义登录页面。云组织 SSO 默认是禁用的。

与标准 SSO 相比，云组织 SSO 提供了更多的灵活性和自定义选项，以便你更好地满足组织的安全性和合规性要求。例如，你可以指定登录页面上显示哪些认证方法，限制允许登录的电子邮件域名，并让你的成员使用采用 [OpenID Connect (OIDC)](https://openid.net/connect/) 或 [Security Assertion Markup Language (SAML)](https://en.wikipedia.org/wiki/Security_Assertion_Markup_Language) 身份协议的身份提供商（IdP）登录 TiDB Cloud。

在本文中，你将了解如何将组织的认证方案从标准 SSO 迁移到云组织 SSO。

> **注意：**
>
> 云组织 SSO 功能仅适用于付费组织。

## 开始之前

在迁移到云组织 SSO 之前，请检查并确认本节中有关你组织的项目。

> **注意：**
>
> - 一旦启用云组织 SSO，就无法禁用。
> - 要启用云组织 SSO，你需要具有 TiDB Cloud 组织的 `Organization Owner` 角色。有关角色的更多信息，请参见[用户角色](/tidb-cloud/manage-user-access.md#user-roles)。

### 决定组织的 TiDB Cloud 登录页面的自定义 URL

启用云组织 SSO 后，你的成员必须使用你的自定义 URL 而不是公共登录 URL（`https://tidbcloud.com`）登录 TiDB Cloud。

自定义 URL 在启用后无法更改，因此你需要提前决定要使用的 URL。

自定义 URL 的格式为 `https://tidbcloud.com/enterprise/signin/your-company-name`，其中你可以自定义公司名称。

### 决定组织成员的认证方法

TiDB Cloud 为组织 SSO 提供以下认证方法：

- 用户名和密码
- Google
- GitHub
- Microsoft
- OIDC
- SAML

启用云组织 SSO 时，前四种方法默认是启用的。如果你想强制组织使用 SSO，可以禁用用户名和密码认证方法。

所有启用的认证方法都将显示在你的自定义 TiDB Cloud 登录页面上，因此你需要提前决定要启用或禁用哪些认证方法。

### 决定是否启用自动配置

自动配置是一项功能，允许成员自动加入组织，而无需 `Organization Owner` 或 `Project Owner` 的邀请。在 TiDB Cloud 中，所有支持的认证方法默认都禁用了此功能。

- 当某个认证方法禁用自动配置时，只有被 `Organization Owner` 或 `Project Owner` 邀请的用户才能登录你的自定义 URL。
- 当某个认证方法启用自动配置时，任何使用该认证方法的用户都可以登录你的自定义 URL。登录后，他们会自动被分配组织内的默认**成员**角色。

出于安全考虑，如果你选择启用自动配置，建议在[配置认证方法详情](#步骤-2-配置认证方法)时限制允许的电子邮件域名。

### 通知成员有关云组织 SSO 迁移计划

在启用云组织 SSO 之前，请确保通知你的成员以下内容：

- TiDB Cloud 的自定义登录 URL
- 何时开始使用自定义登录 URL 而不是 `https://tidbcloud.com` 登录
- 可用的认证方法
- 成员是否需要邀请才能登录自定义 URL

## 步骤 1. 启用云组织 SSO

要启用云组织 SSO，请执行以下步骤：

1. 以具有 `Organization Owner` 角色的用户身份登录 [TiDB Cloud 控制台](https://tidbcloud.com)，然后使用左上角的组合框切换到目标组织。
2. 在左侧导航栏中，点击**组织设置** > **认证**。
3. 在**认证**页面上，点击**启用**。
4. 在对话框中，输入组织的自定义 URL，该 URL 在 TiDB Cloud 中必须是唯一的。

    > **注意：**
    >
    > 一旦启用云组织 SSO，URL 就无法更改。你组织中的成员只能使用你的自定义 URL 登录 TiDB Cloud。如果你以后需要更改配置的 URL，请联系 [TiDB Cloud 支持团队](/tidb-cloud/tidb-cloud-support.md)寻求帮助。

5. 点击**我理解并确认**复选框，然后点击**启用**。

    > **注意：**
    >
    > 如果对话框包含需要为云组织 SSO 重新邀请和重新加入的用户列表，TiDB Cloud 将在你启用云组织 SSO 后自动向这些用户发送邀请电子邮件。收到邀请电子邮件后，每个用户需要点击电子邮件中的链接验证其身份，然后会显示自定义登录页面。

## 步骤 2. 配置认证方法

在 TiDB Cloud 中启用认证方法允许使用该方法的成员使用你的自定义 URL 登录 TiDB Cloud。

### 配置用户名和密码、Google、GitHub 或 Microsoft 认证方法

启用云组织云后，你可以按如下方式配置用户名和密码、Google、GitHub 或 Microsoft 认证方法：

1. 在**组织设置**页面上，根据需要启用或禁用 Google、GitHub 或 Microsoft 认证方法。
2. 对于已启用的认证方法，你可以点击 <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 20H21M3.00003 20H4.67457C5.16376 20 5.40835 20 5.63852 19.9447C5.84259 19.8957 6.03768 19.8149 6.21663 19.7053C6.41846 19.5816 6.59141 19.4086 6.93732 19.0627L19.5001 6.49998C20.3285 5.67156 20.3285 4.32841 19.5001 3.49998C18.6716 2.67156 17.3285 2.67156 16.5001 3.49998L3.93729 16.0627C3.59139 16.4086 3.41843 16.5816 3.29475 16.7834C3.18509 16.9624 3.10428 17.1574 3.05529 17.3615C3.00003 17.5917 3.00003 17.8363 3.00003 18.3255V20Z" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"></path></svg> 配置方法详情。
3. 在方法详情中，你可以配置以下内容：

    - [**自动配置账户**](#决定是否启用自动配置)

        默认禁用。你可以根据需要启用它。出于安全考虑，如果你选择启用自动配置，建议限制允许的电子邮件域名进行认证。

    - **允许的电子邮件域名**

        配置此字段后，只有使用此认证方法的指定电子邮件域名才能使用自定义 URL 登录 TiDB Cloud。填写域名时，需要排除 `@` 符号，并用逗号分隔。例如，`company1.com,company2.com`。

        > **注意：**
        >
        > 如果你已配置电子邮件域名，在保存设置之前，请确保添加你当前用于登录的电子邮件域名，以避免被 TiDB Cloud 锁定。

4. 点击**保存**。

### 配置 OIDC 认证方法

如果你有使用 OIDC 身份协议的身份提供商，可以启用 OIDC 认证方法进行 TiDB Cloud 登录。

在 TiDB Cloud 中，OIDC 认证方法默认是禁用的。启用云组织云后，你可以按如下方式启用和配置 OIDC 认证方法：

1. 从你的身份提供商获取以下信息用于 TiDB Cloud 组织 SSO：

    - 发行者 URL
    - 客户端 ID
    - 客户端密钥

2. 在**组织设置**页面上，点击**认证**标签，找到**认证方法**区域中的 OIDC 行，然后点击 <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 20H21M3.00003 20H4.67457C5.16376 20 5.40835 20 5.63852 19.9447C5.84259 19.8957 6.03768 19.8149 6.21663 19.7053C6.41846 19.5816 6.59141 19.4086 6.93732 19.0627L19.5001 6.49998C20.3285 5.67156 20.3285 4.32841 19.5001 3.49998C18.6716 2.67156 17.3285 2.67156 16.5001 3.49998L3.93729 16.0627C3.59139 16.4086 3.41843 16.5816 3.29475 16.7834C3.18509 16.9624 3.10428 17.1574 3.05529 17.3615C3.00003 17.5917 3.00003 17.8363 3.00003 18.3255V20Z" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"></path></svg> 显示 OIDC 方法详情。
3. 在方法详情中，你可以配置以下内容：

    - **名称**

        为要在自定义登录页面上显示的 OIDC 认证方法指定名称。

    - **发行者 URL**、**客户端 ID** 和**客户端密钥**

        粘贴从你的 IdP 获取的相应值。

    - [**自动配置账户**](#决定是否启用自动配置)

        默认禁用。你可以根据需要启用它。出于安全考虑，如果你选择启用自动配置，建议限制允许的电子邮件域名进行认证。

    - **允许的电子邮件域名**

        配置此字段后，只有使用此认证方法的指定电子邮件域名才能使用自定义 URL 登录 TiDB Cloud。填写域名时，需要排除 `@` 符号，并用逗号分隔。例如，`company1.com,company2.com`。

        > **注意：**
        >
        > 如果你已配置电子邮件域名，在保存设置之前，请确保添加你当前用于登录的电子邮件域名，以避免被 TiDB Cloud 锁定。

4. 点击**保存**。

### 配置 SAML 认证方法

如果你有使用 SAML 身份协议的身份提供商，可以启用 SAML 认证方法进行 TiDB Cloud 登录。

> **注意：**
>
> TiDB Cloud 使用电子邮件地址作为不同用户的唯一标识符。因此，请确保在你的身份提供商中为组织成员配置了 `email` 属性。

在 TiDB Cloud 中，SAML 认证方法默认是禁用的。启用云组织云后，你可以按如下方式启用和配置 SAML 认证方法：

1. 从你的身份提供商获取以下信息用于 TiDB Cloud 组织 SSO：

    - 登录 URL
    - 签名证书

2. 在**组织设置**页面上，点击左侧导航栏中的**认证**标签，找到**认证方法**区域中的 SAML 行，然后点击 <svg width="16" height="16" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M12 20H21M3.00003 20H4.67457C5.16376 20 5.40835 20 5.63852 19.9447C5.84259 19.8957 6.03768 19.8149 6.21663 19.7053C6.41846 19.5816 6.59141 19.4086 6.93732 19.0627L19.5001 6.49998C20.3285 5.67156 20.3285 4.32841 19.5001 3.49998C18.6716 2.67156 17.3285 2.67156 16.5001 3.49998L3.93729 16.0627C3.59139 16.4086 3.41843 16.5816 3.29475 16.7834C3.18509 16.9624 3.10428 17.1574 3.05529 17.3615C3.00003 17.5917 3.00003 17.8363 3.00003 18.3255V20Z" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"></path></svg> 显示 SAML 方法详情。
3. 在方法详情中，你可以配置以下内容：

    - **名称**

        为要在自定义登录页面上显示的 SAML 认证方法指定名称。

    - **登录 URL**

        粘贴从你的 IdP 获取的 URL。

    - **签名证书**

        粘贴从你的 IdP 获取的完整签名证书，包括开始行 `---begin certificate---` 和结束行 `---end certificate---`。

    - [**自动配置账户**](#决定是否启用自动配置)

        默认禁用。你可以根据需要启用它。出于安全考虑，如果你选择启用自动配置，建议限制允许的电子邮件域名进行认证。

    - **允许的电子邮件域名**

        配置此字段后，只有使用此认证方法的指定电子邮件域名才能使用自定义 URL 登录 TiDB Cloud。填写域名时，需要排除 `@` 符号，并用逗号分隔。例如，`company1.com,company2.com`。

        > **注意：**
        >
        > 如果你已配置电子邮件域名，在保存设置之前，请确保添加你当前用于登录的电子邮件域名，以避免被 TiDB Cloud 锁定。

    - **SCIM 配置账户**

        默认禁用。如果你想从身份提供商集中和自动化 TiDB Cloud 组织用户和组的配置、取消配置和身份管理，可以启用它。有关详细配置步骤，请参见[配置 SCIM 配置](#配置-scim-配置)。

4. 点击**保存**。

#### 配置 SCIM 配置

[跨域身份管理系统（SCIM）](https://www.rfc-editor.org/rfc/rfc7644)是一个开放标准，可自动化身份域和 IT 系统之间的用户身份信息交换。通过配置 SCIM 配置，可以将身份提供商的用户组自动同步到 TiDB Cloud，你可以在 TiDB Cloud 中集中管理这些组的角色。

> **注意：**
>
> SCIM 配置只能在 [SAML 认证方法](#配置-saml-认证方法)上启用。

1. 在 TiDB Cloud 中，启用 [SAML 认证方法](#配置-saml-认证方法)的 **SCIM 配置账户**选项，然后记录以下信息以供后续使用。

    - SCIM 连接器基本 URL
    - 用户唯一标识符字段
    - 认证模式

2. 在你的身份提供商中，为 TiDB Cloud 配置 SCIM 配置。

    1. 在你的身份提供商中，为 SAML 应用集成添加 TiDB Cloud 组织的 SCIM 配置。

        例如，如果你的身份提供商是 Okta，请参见[为应用集成添加 SCIM 配置](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_scim.htm)。

    2. 将 SAML 应用集成分配给身份提供商中的所需组，以便组中的成员可以访问和使用应用集成。

        例如，如果你的身份提供商是 Okta，请参见[将应用集成分配给组](https://help.okta.com/en-us/content/topics/provisioning/lcm/lcm-assign-app-groups.htm)。

   3. 将用户组从身份提供商推送到 TiDB Cloud。

        例如，如果你的身份提供商是 Okta，请参见[管理组推送](https://help.okta.com/en-us/content/topics/users-groups-profiles/usgp-group-push-main.htm)。

3. 在 TiDB Cloud 中查看从身份提供商推送的组。

    1. 在 [TiDB Cloud 控制台](https://tidbcloud.com)中，使用左上角的组合框切换到目标组织。
    2. 在左侧导航栏中，点击**组织设置** > **认证**。
    3. 点击**组**标签。显示从身份提供商同步的组。
    4. 要查看组中的用户，点击**查看**。

4. 在 TiDB Cloud 中，为从身份提供商推送的组授予角色。

    > **注意：**
    >
    > 向组授予角色意味着组中的所有成员都获得该角色。如果组包含已在 TiDB Cloud 组织中的成员，这些成员也会获得组的新角色。

    1. 要向组授予组织角色，点击**按组织**，然后在**组织角色**列中配置角色。要了解组织角色的权限，请参见[组织角色](/tidb-cloud/manage-user-access.md#organization-roles)。
    2. 要向组授予项目角色，点击**按项目**，然后在**项目角色**列中配置角色。要了解项目角色的权限，请参见[项目角色](/tidb-cloud/manage-user-access.md#project-roles)。

5. 如果你在身份提供商中更改了推送组的成员，这些更改会动态同步到 TiDB Cloud 中的相应组。

    - 如果在身份提供商的组中添加了新成员，这些成员将获得相应组的角色。
    - 如果从身份提供商的组中删除了某些成员，这些成员也会从 TiDB Cloud 中的相应组中删除。
