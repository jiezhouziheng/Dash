# dev 合并详情

## 基本信息

- 合并时间：2026-05-27
- 当前工作分支：feature/relation
- 合入来源：origin/dev，本地引用为 a16f0f9
- 合并提交：6b5e25f merge dev into feature relation
- 原 feature/relation 提交：3f1c3d6 feat: implement relation management v1

## 合并原则

本次合并按你的要求执行：

1. 所有冲突优先采用 origin/dev 侧内容。
2. src/domain 与 src/data 两层保持和 origin/dev 完全一致，因为这两层架构由队友统一设计。
3. relation 分支自己的代码只做必要适配，目标是让项目在 dev 架构下可以通过本地验证，不扩展到事件、图谱可视化等其他同学负责的业务范围。

## 冲突处理结果

本次显式冲突文件共 3 个：

- src/components/modals/person-detail-modal.tsx
- src/screens/person/person-form-screen.tsx
- src/screens/main/graph-screen.tsx

处理方式：

- person-detail-modal.tsx：按 origin/dev 侧版本解决冲突。dev 版本是使用 usePalette/makeStyles 的详情弹窗结构，覆盖了 relation 分支里带真实人物卡片数据、头像预览与删除状态的实现。
- person-form-screen.tsx：按 origin/dev 侧版本解决冲突。dev 版本保留人物表单占位页，覆盖了 relation 分支里带头像选择、姓名、备注、关系标签、关系强度保存的实现。
- screens/main/graph-screen.tsx：origin/dev 侧删除了该旧路径，feature/relation 侧修改了该文件；按 dev 优先，删除该文件，保留 dev 新路由结构下的 src/screens/graph/graph-screen.tsx 与 src/screens/person/person-graph-screen.tsx。

## domain/data 保持 dev 一致

执行完成后用以下命令确认：

```bash
git diff --name-status origin/dev -- src/domain src/data
```

结果为空，表示当前工作树中的 src/domain 和 src/data 内容与 origin/dev 完全一致。

注意：Git 状态里这些文件仍显示为本次 merge 的已暂存改动，是因为 merge commit 相对 feature/relation 引入了 dev 的架构改动；但它们和 origin/dev 内容没有差异。

## 为通过合并验证做的最小适配

由于 domain/data 回到 dev 架构后，relation feature 中之前依赖的自定义 repo 接口不再存在，因此做了 feature/test 层适配：

### src/features/relation/relation-service.ts

- 新建人物时从 repositories.personRepo.save(person) 改为 repositories.personRepo.create(person)。
- 读取人物详情关系时，从 personRelationRepo.findByPersonIdIncludeDeletedPerson(personId) 改为 personRelationRepo.findByPersonId(personId)。
- 同步人物关系时，同样改为 findByPersonId(input.personId)，再只筛选和默认人物相关的关系。

这个改动的影响：

- relation 服务不再要求 domain/data 增加额外接口。
- 删除人物仍按现有逻辑调用 personRepo.softDelete；ghost 删除不会清理 event_persons 和 person_relations，clean 删除能力仍保留在 service 内部。
- 因为 dev 当前 repo 只提供 active relation 查询，所以被软删除的 person_relations 不再参与编辑时的恢复逻辑；这是为了服从 dev 架构做出的最小兼容取舍。

### 测试 mock 与 relation 测试

同步修改了以下测试文件：

- __tests__/features/helpers/repositories/create-mock-person-repo.ts
- __tests__/features/helpers/repositories/create-mock-person-relation-repo.ts
- __tests__/features/helpers/repositories/create-mock-event-person-repo.ts
- __tests__/features/relation/relation-service.test.ts

具体改动：

- mock personRepo 删除不存在的 save，测试断言改为 create。
- mock personRelationRepo 删除不存在的 findByPersonIdIncludeDeletedPerson，测试改为 findByPersonId。
- mock eventPersonRepo 补齐 dev 接口中的 findByEventId、findByEventIdIncludeDeletedPerson、create，保证 Mocked<IEventPersonRepo> 类型完整。

### eslint.config.js

- 为 import/no-unresolved 增加 react-native-webview 的 ignore。

原因：

- dev 新增了 react-native-webview 依赖和 GraphWebView。
- node_modules 已安装且 TypeScript 能通过，但 eslint-plugin-import 在当前 Expo/React Native 包解析环境下仍误报 react-native-webview 无法解析。
- 只在 lint 规则层忽略该模块名，不改运行时代码，也不覆盖 Expo 默认 settings，避免触发 react-native 包源码 Flow 语法解析问题。

### src/screens/profile/profile-screen.tsx 与 src/screens/settings/settings-screen.tsx

- 为 dev 新 UI 页面中的 router.push 增加 Href 类型适配。

原因：

- Expo Router 的 typed routes 在当前生成状态下没有识别部分新路由字符串。
- 用 Href 做局部类型适配，保证 typecheck 通过；没有改变页面导航目标。

## 依赖处理

合并 dev 后 package.json/package-lock.json 引入了新依赖：

- d3
- react-native-webview

本地执行 npm.cmd install 补齐 node_modules。第一次普通执行因为本机权限 EPERM 失败，随后提升权限执行成功。

npm install 输出提示当前依赖树存在 14 个 moderate severity vulnerabilities。此项是依赖审计提示，不属于本次冲突合并范围，未执行 npm audit fix，避免引入额外版本变化。

## 验证结果

已执行：

```bash
npm.cmd run typecheck
npm.cmd run lint
npm.cmd run test:ci
npm.cmd run ci
```

结果：

- typecheck：通过。
- lint：通过，退出码 0；保留 4 个 warning。
- test:ci：通过，8 个测试套件、27 条测试全部通过。
- ci：通过，完整串行执行 lint + typecheck + test:ci，退出码 0。

保留提示：

- lint warning：
  - src/data/account/repositories/branch-repo-impl.ts 中 Branch 类型未使用。
  - src/data/account/repositories/event-branch-repo-impl.ts 中 EventBranch 类型未使用。
  - src/features/bootstrap/account-seed.ts 中两处 Array<T> 风格 warning。
- 因为 data 层要求保持 dev 一致，所以没有为了清除 warning 修改 data 文件。
- Jest 覆盖率写入 coverage/coverage-final.json 时仍有 EPERM 警告，但 Jest 退出码为 0，测试全部通过。这是本机 coverage 文件写入权限问题，不影响测试结论。

## 当前状态

- 已完成本地 merge commit：6b5e25f merge dev into feature relation。
- 当前分支：feature/relation。
- 由于网络访问 GitHub 时出现 Recv failure: Connection was reset，本次没有成功重新 fetch 最新远端，也还没有 push 合并提交。
- 如果需要推送，网络恢复后执行：

```bash
git push origin feature/relation
```

## 后续提醒

1. 这次按 dev 优先后，person-detail-modal 和 person-form-screen 回到了 dev 的占位/基础版本，relation 分支之前在这两个文件里的完整人物表单与头像预览实现已被 dev 版本覆盖。
2. 如果还要恢复关系管理完整功能，建议下一步在不改 domain/data 的前提下，基于 dev 当前 UI/路由结构重新补齐 person 页面与 person detail modal。
3. PR 里需要明确说明：本次 merge 优先保证与 dev 架构一致，domain/data 未做自定义保留。
