# Skywalking Injection

# 前言
官方资料：[https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks)

JsonPach：[http://jsonpatch.com](http://jsonpatch.com)

基于Kubernetes Admission Controllers的Java Agent自动挂载方式

目前仅支持Java Agent的挂载

# Kubernetes Admission Controllers
Kubernetes 准入控制器是控制和强制使用集群的一种插件。我们可以把它看作是拦截（已认证）API 请求的拦截器，它可以更改请求对象，甚至完全拒绝请求。<br />Kubernetes
 准入控制器的“准入控制链”分为两个阶段：变更（Mutating）准入控制，修改请求的对象；验证（Validating）准入控制，验证请求的对象。因此准入控制器即可以被用作变更和验证，也可以两者结合起来使用。<br />![ztrwd7qfd7.jpeg](https://cdn.nlark.com/yuque/0/2020/jpeg/87651/1596552543132-b3d3c55a-9863-4ed2-ab99-5c69a0e31a29.jpeg#align=left&display=inline&height=402&margin=%5Bobject%20Object%5D&name=ztrwd7qfd7.jpeg&originHeight=402&originWidth=1080&size=25626&status=done&style=none&width=1080)

# 基本思路
通过变更控制器的Webhook方式，在发布时对POD的配置进行变更，从而挂载Skywalking Java Agent与启动命令。<br />其挂载方式使用InitContainer的方式，启动方式则可使用增加环境变量的方式。<br />最终生成的pod配置示例：
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
    skywalking: enabled
    skywalking-volume: sw-volume-09befabe
  name: nginx
  namespace: skac
spec:
  containers:
    - env:
        - name: SWKAC_ENABLE
          value: "true"
        - name: JAVA_TOOL_OPTIONS
          value: -javaagent:/opt/skywalking/skywalking-agent.jar
        - name: SW_AGENT_COLLECTOR_BACKEND_SERVICES
          value: apm-aop.apm:11800
        - name: SW_AGENT_NAME
          value: pod-nginx
      image: nginx
      imagePullPolicy: IfNotPresent
      name: nginx
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: default-token-bptdb
          readOnly: true
        - mountPath: /opt/skywalking
          name: sw-volume-09befabe
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  initContainers:
    - env:
        - name: AGENT_HOME
          value: /opt/skywalking
      image: ilemontech/skywalking-java-agent
      imagePullPolicy: Always
      name: sw-init-09befabe
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
        - mountPath: /opt/skywalking
          name: sw-volume-09befabe
  volumes:
    - name: default-token-bptdb
      secret:
        defaultMode: 420
        secretName: default-token-bptdb
    - emptyDir:
        sizeLimit: 200Mi
      name: sw-volume-09befabe
```
# 操作步骤
本示例中所有内容均部署在namespace: skywalking中
## 准备证书
> 调用时会默认使用https方式调用。
> 可以自行创建CA与证书文件。

示例：
```yaml
CA: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZvakNDQTRvQ0NRREZYZTRwM1o0NjREQU5CZ2txaGtpRzl3MEJBUXNGQURDQmtURUxNQWtHQTFVRUJoTUMKUTA0eEVEQU9CZ05WQkFnTUIwSmxhV3BwYm1jeEVEQU9CZ05WQkFjTUIwSmxhV3BwYm1jeER6QU5CZ05WQkFvTQpCbWxNWlcxdmJqRVBNQTBHQTFVRUN3d0dhVXhsYlc5dU1Sc3dHUVlEVlFRRERCSjNkM2N1YVd4bGJXOXVkR1ZqCmFDNWpiMjB4SHpBZEJna3Foa2lHOXcwQkNRRVdFR3hwY0dkQWIzVjBiRzl2YXk1amIyMHdJQmNOTWpBd056RTMKTURFME5EQXhXaGdQTWpFeU1EQTNNVGN3TVRRME1ERmFNSUdSTVFzd0NRWURWUVFHRXdKRFRqRVFNQTRHQTFVRQpDQXdIUW1WcGFtbHVaekVRTUE0R0ExVUVCd3dIUW1WcGFtbHVaekVQTUEwR0ExVUVDZ3dHYVV4bGJXOXVNUTh3CkRRWURWUVFMREFacFRHVnRiMjR4R3pBWkJnTlZCQU1NRW5kM2R5NXBiR1Z0YjI1MFpXTm9MbU52YlRFZk1CMEcKQ1NxR1NJYjNEUUVKQVJZUWJHbHdaMEJ2ZFhSc2IyOXJMbU52YlRDQ0FpSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0lQQURDQ0Fnb0NnZ0lCQUxIUS9ndTE0WUJDYjNRNSthTWtyT3dEeUpQKzQvQUpjRTg0Yk5tci9WbFhEMW9JCnMzTDJvclFicGtRQ1VEVmZqVHFOTWtvT3QrbVpYWHdtMnUzMG1TTkpCRmhaeHEzT2dLSVZxV2IxVWQ5WDRBMjMKOUlxSWREV0NOMndVWlhVd0ZuTTdCYW5xUyt3cmJob3ZxalNFcUI2OUJpbkdjamZvTCsxaWUzdVhqeVBHZklWagpieldVSUdnYzNpVEllc2xyL1prUzBLaDYzbFVtN0NaSFBacVpwMy9vVVNYQmszTXN4dkpCdFZ4QlRQNG5FMCtiCmYvWEkveElTby91ckI1a2RYZW1RM2lnRUtRQmpTWXAvbkthdWEvUHJrZ3BHbVl1a2RlWElVeXhKQk9rNUJCdXcKb3Fid0NBcEFpQ0wzeUMvZnVreEdPZjd6dHdiRGZYU2hXT3ZTSks2NFd0Y1owRHhsQ05VTDFOZzZrcjFLMjRNeApHazZjRDViUWJjWCtEVzhTS25iS0NabzhLR1Jvdjl1cG9tRVc0Q2kzWE8rTjF6cGRST2lwYVpXVTNLdnJGZ2E2CjY5ZmhJM0dnM2RVbFprK2hUazVkMUExNysxU3l3QXRRaldxeG93R3ljY1pPSnRqOUpZaUlITXJsRU9SeWZQTzIKNVg0UG1VSmtoSW9BYWNSdm1oaWRTRFlRTmNMZUt4Ym1TVXdWZEhaSkpGWjJaVjFpcnZPMmM4VEtqNlBXZDA3YQpGNTRvK0JLMVQyMjFWK0hnUnJWbTlaNE5zQWNNc1BScHYzYWNCdk1aWWxxL0h4amttU1NNSHFtd0UwY1d1c28yCkZxdmNiMXBKUGJMdGpQZ1d3VmRPblZxTUc3aTEzYU5uck1mODVKc2VzZVp6VE9zLzFCdlhyODUxVTVMZkFnTUIKQUFFd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dJQkFJWjdaWFpXNEhKMTI3enh2T1ltSHU3aC9rNUdwb294dkorcApZS29QRjRpd3BGRGM3ZjBQZHk0K0tVcG1RS2xjYVJuSEZWWGQ5WEo0R1lnNjg5a0xUeVo3c29iam5RdDZOU0tOCmxYTmdvSUp3Zm1ONXpqV25HUCtXTWhvQlJMM0o0bjNTWUdtalkwZTRzOU1wUlBRdklTeUdZcytFQUQxVGozZEQKdnFoM05WTnd2bjJsNkkyY0lxUEVoU01SY3QwU3lPc2o4MDIveFBvajZtZStqRjk4ZGc2V1M5VUhBSnNqcUV6WApzbHdhQTl6c3ZwQlo0TjVjdXNRMmpHWEdEbkVDQUtMNm1idnRVWHBYd012SFd5MG1vQ1g4c0tKcEJkNlZiUHhMCnhlT1duTnpDRWQvYXdqZVk1N3BzOTNidjZVejZ1a2UyV2ZiOUdMd1BCRmQ5RUs1NkV5Y2U5d2daSThvaDJOSVkKYjY2RHAwbExhR2k0OGRZWkpGRU1ETFhOVUV2RXRmZEo3K3lsU1cyWG52bkxDT25oQjJiQm93YmxlMDc1UllzagpsdFVKRWw1YTFZbUhOWnMzeGdyd1ErL3N4bjg1YlArempTSUlqUms1V1Bpbm4vL25vdGp6UHJJSGJmTjY0by95CjAvVWlrZVRjVFZuZUREQUJwdGZ5VXlEbU5pVlhCaHlodks3MTJEZmZ1Q01aclQ1YlF6VUd1cytBUGo4QThoZ0kKbXFSd25LdnJvcHJ2OEp3b2VZbm1rNExkZmxMSlFYczJVdWd2L1p1VmVkeTFBSk5rZThUSXhhRUoyS2hTMGdacQowWXZEVkFRRUg0VlRMeDhWRTdxRm42QTVhaERtVUF0ZEJ5MUNkSy9JQ1o4eDdKZkRCYS9zK0p1K1ZBTGt5Mm9hCjRxVlNEUHQyCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
TLS.CRT: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZzekNDQTVzQ0NRQ3h5Q2xkRVVFd2RUQU5CZ2txaGtpRzl3MEJBUVVGQURDQmtURUxNQWtHQTFVRUJoTUMKUTA0eEVEQU9CZ05WQkFnTUIwSmxhV3BwYm1jeEVEQU9CZ05WQkFjTUIwSmxhV3BwYm1jeER6QU5CZ05WQkFvTQpCbWxNWlcxdmJqRVBNQTBHQTFVRUN3d0dhVXhsYlc5dU1Sc3dHUVlEVlFRRERCSjNkM2N1YVd4bGJXOXVkR1ZqCmFDNWpiMjB4SHpBZEJna3Foa2lHOXcwQkNRRVdFR3hwY0dkQWIzVjBiRzl2YXk1amIyMHdJQmNOTWpBd056RTMKTURFMU16VTFXaGdQTWpFeU1EQTNNVGN3TVRVek5UVmFNSUdpTVFzd0NRWURWUVFHRXdKRFRqRVFNQTRHQTFVRQpDQXdIUW1WcGFtbHVaekVRTUE0R0ExVUVCd3dIUW1WcGFtbHVaekVQTUEwR0ExVUVDZ3dHYVV4bGJXOXVNUTh3CkRRWURWUVFMREFacFRHVnRiMjR4TERBcUJnTlZCQU1NSTNOcmVYZGhiR3RwYm1jdGFXNXFaV04wYVc5dUxuTnIKZVhkaGJHdHBibWN1YzNaak1SOHdIUVlKS29aSWh2Y05BUWtCRmhCc2FYQm5RRzkxZEd4dmIyc3VZMjl0TUlJQwpJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBZzhBTUlJQ0NnS0NBZ0VBeCt1V0NRaDJWRTZtYVF5N0FFa0IxdnU1ClV1blJDWjlNcUpHY284c0lkQ2plOGN4WE5QbXdyMXVzOFEyVVAvcUZYWkw5VUJ6R2gyV3FaSTdRc25Dbm5YcVYKcUpyS0IrR3kvTncvWnFEV1hJQnhSYTBsZEdicEkxbjdVSzVyZWJENTJjVGhMeVBQcWtBV2I3ejJRYnFEbTRheQpsV3FwN0h2MThDUHlidklJMUc2MHorUFR1WTdWc2hwTEJhQ0RXdy9pOUdXTCtBWnpSYjdlcnBJeWYxdGtaNjY3CmhlM21aUDc0cklHaVJsK1NHT3BxNnVYQUYwWXRsTFdPMHBCY2ZlaUxlTm9PcUVXUmRqMDVHUkg0eHE2bzZ2YzQKZlNlUzRvYy94R0wwTnVYV0ZENitaZEE2V0tJYm8rQ2wyd21abzFQRXVFcTRxUkJhQXpLZm1EUThrRXpEc28zeQoxb2VwUUVyVFVBUFFqYVZrOThjcU9ROEtIbWEyaUZLOTBoc0IvS0NHNXlpTUF3V0JhckdRTzhuQm5yejZFeEN3CjM2ZlZ4eVpmazloY3R3TzZwWTNibWJuM0ExaVMwNERzNUxhdFZaM3QrN2c5VHJ2Tm91anBHdTJLblhrcU5oODIKa21TKzZ4UEZtZ1Z2dUx6NGtpS3oxbDdmaThzVzFQd2RBdHFKQkJsZDY4MHA3ek5yeFp4MTlVNjVobkl5dVh6Wgo1WHZSYzI4RUNhWVFhNTQybS9rVzU0YkN1QUo1TkRmVnVhWGNiZThrN2hlYk1RaGF1Zk4xRy8rTTJMVXJvczdCCi9aRnFYdWtjQ3F1KzczdWRTMkJCM2pWeG9FTTV3ajNZM0ZpYUptSUh4blhOL2VzcFZ5dHZNU3hQVVpjdWlieWwKSGVLRDcrZEsvK2dxcFhsd3Awc0NBd0VBQVRBTkJna3Foa2lHOXcwQkFRVUZBQU9DQWdFQUhzeDcySjVpMjdTZwpqb2g2N2hnbDBpZUt3ajdrVnljM2JWUmZvQS83NTgweUVQMjl2MWMwbksyVktpYzYrcHM0eGt2WHRpKzVsNVdlCmpmb0p1TmFFVVNHN0xTNVBZUm12aUJQZEVXY2ZteWdRYzhTcjdYeFVreVJyaGFuMnFLWGZwZkVMMmh3YVl1N2wKano2OEdjS0dPL3RJTUphWFRZWHVNQm9SeTdZY0JzY2tXejlIcTZjYndkZXhhdEo5eUhjenF5ZzBUUXZrRXBmMQpSck1DWE0yZHp1Wm1neHFmNkhzamp3Zm9KcWJGS3IrZDBBNjlBUGNJT0N3QUlaSU9VNzIxNko1Mms0N3JINnVaCmRGdWR4QUU5UExSWVVsQTQ2S1RKemVETmxrZGxRSnNQU05QclB0MTdGTmpNbTBNN1dLWDh0ZW9BUHNSWDkwZm4Kd203Qlc3Z2RwQVJ0NGhWdWQ2cmxycmlZWUpKQXNZRjNHV1lGT2RuaVJraHVycGRuUmgza1ZrZU53bDhOb3ZmTgpoY2tnQ0Rvb09jMHZpNHNvNXBwMUFSWHlMWUFsUnlaU2NHQ3U5WTJsUkwvTGJOb1JlRGgrcEJNbGtSa2J5S3ZyCkpGYUZxbVIvVFhXVkRwVFNDNGhQcmkzOVE3eDRJVDRUQUpZQjAvM2FTT1Nlczl0YXZSeUhWREFWR3MvSzJxaDAKeEpaK0c5NVZqQ1ZjWTU2NDBuNzZycDNZZEVlMXdteU9weWhCOSthZ0JMZGlxSVMzQ3ppbUJJeEhRc3hsdXF6Zwo1Ky9Ha1crZTh0TDBKRnVKZ2hkSE1wTzZkaW5XMDVCN2xadWlVSisxS2lsTWZZVmQ0K1poTlp6bDRmcGZhazg2CmxmdXg2UzNOSktVcFQzU3dIRW5ZOXBoRmVsLzlHMms9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
TLS.KEY: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlKSndJQkFBS0NBZ0VBeCt1V0NRaDJWRTZtYVF5N0FFa0IxdnU1VXVuUkNaOU1xSkdjbzhzSWRDamU4Y3hYCk5QbXdyMXVzOFEyVVAvcUZYWkw5VUJ6R2gyV3FaSTdRc25Dbm5YcVZxSnJLQitHeS9Ody9acURXWElCeFJhMGwKZEdicEkxbjdVSzVyZWJENTJjVGhMeVBQcWtBV2I3ejJRYnFEbTRheWxXcXA3SHYxOENQeWJ2SUkxRzYweitQVAp1WTdWc2hwTEJhQ0RXdy9pOUdXTCtBWnpSYjdlcnBJeWYxdGtaNjY3aGUzbVpQNzRySUdpUmwrU0dPcHE2dVhBCkYwWXRsTFdPMHBCY2ZlaUxlTm9PcUVXUmRqMDVHUkg0eHE2bzZ2YzRmU2VTNG9jL3hHTDBOdVhXRkQ2K1pkQTYKV0tJYm8rQ2wyd21abzFQRXVFcTRxUkJhQXpLZm1EUThrRXpEc28zeTFvZXBRRXJUVUFQUWphVms5OGNxT1E4SwpIbWEyaUZLOTBoc0IvS0NHNXlpTUF3V0JhckdRTzhuQm5yejZFeEN3MzZmVnh5WmZrOWhjdHdPNnBZM2JtYm4zCkExaVMwNERzNUxhdFZaM3QrN2c5VHJ2Tm91anBHdTJLblhrcU5oODJrbVMrNnhQRm1nVnZ1THo0a2lLejFsN2YKaThzVzFQd2RBdHFKQkJsZDY4MHA3ek5yeFp4MTlVNjVobkl5dVh6WjVYdlJjMjhFQ2FZUWE1NDJtL2tXNTRiQwp1QUo1TkRmVnVhWGNiZThrN2hlYk1RaGF1Zk4xRy8rTTJMVXJvczdCL1pGcVh1a2NDcXUrNzN1ZFMyQkIzalZ4Cm9FTTV3ajNZM0ZpYUptSUh4blhOL2VzcFZ5dHZNU3hQVVpjdWlieWxIZUtENytkSy8rZ3FwWGx3cDBzQ0F3RUEKQVFLQ0FnQjZqYm5RMmNZb0VuckM1Ukh4cjgrWDZZSUhDTnlqS0d0RzN0U0dPQjE3ajk1em5jOUw0MURqZnJCNgpxTmRialFpYWYxUVgrenZjbm44ZzI4ZWx5NWVFMnpQSlF2SmNlSWhQcC9pWndqa0dSWnFxbXVUZUFZUWpycko0CkdDUThXamdmdmloSXpJd3VFdEZ1T0FsQjY4Tm40Tit5NXNzNTlCZWsyWFJ4ZkovYkVlNTNqbk1wU2gxSDdBUlgKYzdseVpqKzdrRkNJZi9QRUtxaTVuVmVsYVBJWStTMXRnWlArRDZ1RG5JaE0yYU5tOExVUVFEMU1SeklEK25TTwpNZjd4RkZBRGlyRDNpSVlPVktPZThieHVsMzFuWkp2VlpTQVBDOWdVWEpEVzRuZ2t1djNlemhJbWFXSEdTUjJECk05UmlJaVR5MEtONHlGUlZwMy9jUHhGdGFCR0NXc2ZTM0U4endyellCWkxDK3E5aHQ1REhWRDFFZVVja0wyVEsKc1IvOEJRaGtqd3A4UnI1NmJqVURGSHg3REhUQ0xia3RObEdIRS9xZng3bzZLME85UDMrcVJNUFJkVExsTUNsWApGbW03NEFKNnFlVlZMelZZSXBCMzI1UEpmZHFHb3NXWEZGd09IZEZLMWF3ZGU3VTZTcjV1dWkxOHROTTA2dzY2CjJQL2NBRWVveXlPYktjblJRR3RUOEh3M1Roem1VTGxVVnhBV0c5QytKS2YyQU84Z1Rwc3NFeGdCTmRid244UlIKUCtkbHRXdjhzczZkQzNoWld4OHFPbG1id09VRFBDd2pwSytjSUhyT21XS3F3WW1uc1dhLzhtRkxGTVdFbE9CWgprMGwwa0RhcTVrQmVRRDRkT0ozZ1RybVRSVGVGS20wNVk2OHk2Wm4yZEpKZFUyK29vUUtDQVFFQThNM2lPeUhaCk1CM3FFUmROeFBJQmxOdExFbVp1ck1qVkJ2MVFrUG91eTVPdlRlTjI0UnhVZzZ0ZjZONlIraVVJOGFIcDdvRFkKZzYvVW43UUFldGNhZ05sd3dZSTAyL3AyOEZrdlF3NXdCZy9VNmpIOEVNMnFiWGtxUFNhVTQ5KzJpTWxldytkRApqTFNUaDlaSE5tTWpIMU1iRXlUSjlvYU9MZk55NTVzZS9wNUdsRi9ONm02aWlyci9oYTlMakszMlpodDBVQmJLClEvQi9oeTFCUHNRU1ViL2dXczVsNjduQXgrWmxHSmQzMkJnc1pYZ1pJc2JybVdxQWdCQXpEOTcxUDVZL3hZSEcKaEpYV3IzS2kxaVZOMHJiYjNYS3JuWU5XWVpXVDlGc1JveGRkNmlBNGIwcHUxYkRHUDgwS2dqVmtMaHQ4Ung2aQppbzNGb0lUYW9HQS9ld0tDQVFFQTFJazhFdHhlUVB0cEI1TVltajYrK0R4clZ0SDdVbGlEWisrZTliVnEvbVhwCm9KQS9uaTR3ajlaSXRneklxYUFUcUVZQXJrd0NPRno1Z1VBTk5JZG1KZm1YVFJQWE1OOThGL21QS0NHYzIycVkKWVJGM0V5ZThvc01jNFNDRG13dUFIdmVLbUdla1ovU0lESUE3L3FGRjVvT1BBeFlCdFhmYWw1NkY4S2dEQ2pJdApiVjZwaWFXWXRibldHUktrZ1lub2MzUkZLWlhjVitsWm1DclVIVFZZUFZTTklaRTdjeCtnOThuMUtvZDdlcFAwCkF1MWU5VW5iMDJVcWNNWlR4Z0pWWmxlZ20yQjdTTVNIUGhIRG9iYUJnQVMvRDBhV2psL0NHc1IwbjIvVnlQZWoKMkZpeHEwYnBEb1Avb0ZGMFdjNWt5YVk1WXBxR1ZJN2pwQjc2WVB0R2NRS0NBUUJtSUx1SkhyTHhVRTZjVHdqaAp0VTNzWnhFd3paYTAyenhXQ1l0bzRXS0ZYcTVqMTB2czcwbjh5ZXNVWllqOTBVVTR6ZFhjSEhNM2JwUTc0MmlzCnNKd0I5N1oxcTVabGtaVnVIRFVLT0k1VVU5U1dPdnJQMlpvbGNkVHpPSEMvTUh0SENqaVIxM3h6cVlaRWZ5R1cKdEhjSjZ3STVuVW0ycjV5UGFvZmFLSlQ5b2YwSElwYS9qWXFNVWdEYkh0eG8zc0w0elQ0SVZaSVZQR09wM2d5awo1UjFyNUhmdnEwZllzT0JOSjY3S3Zqckt3SDNienhMQ0NDR1Y0SEI5WCtGNElyUmE4b3M3WmUzZS8zbWMwWDYwCldmL21sVDBzYjF6UG9nczV1ZW1WSWlhWngraEJ5c2FFbE1uckFWMGpZLy9IOTRMMGR4Mi9RQVBweFhVQ3ZqWEcKL09zUEFvSUJBQ2w2VnVUL2FNMlp3eGVkVjNIU3lRWTlGRjJ2UVRNQzN6SkxpMlE4Y1pNUVpKWFhVUE1YNUpaTgpwWWNSMjU5dkJDY1U4bEVSdUdVemorWHZrQ1F1UUhzc3lLQmNQT1RtMkpyOU16a0x4a2J6K0tVSStZb2RlU3p5Cmt4SkVmQ3FvQmVzRk9HRWg2TzM0eFF4YmdsbmhNM2tYTjdQTWVhem1wUnh0N1luWDRDLy9OdXBmVFFyMTdHV2EKelBPN3Rhc3FOTmxEV2JRUjhSeWZCOE13cUpEQ3RtUFJ1bkZRbkorTkRxY05IUEdkMnpWQzVwNGpvU0tSdFhyKwpCVmhZL0VScWRDd3k4YjhoQnk1ZVhGNTZyUWxFU3VXbkIzcDl0aWcyKy9TcFhuRTNMQ3U2V0ptQ1Jibzlib0pNCnRIaWZMT3ZSNGMyUXlTWTdMZmdXb3NxRzJTcFFINkVDZ2dFQUpCdERxd0NReU5NdTZwQ3FGelNNc212TnR1cXUKSmlXSEUrNFdQRTBZd3IwWCtzeFdlUWVBdUo2bGNlWTZFd3Urb0ZZYnJZakVQSUNPVEgyVnZRZUNGWXZSTk9uWQpCdE0xcWQzYTVoWVpPdjhTM1ZWVjNlcHNmR2NBSUxQOTJVR1FITlBoTWZuNGdjaXJla0w5V1FWOUFKNkFLN0tBCjVDdTMwcGJyMTQ0d2NiaVBTc2VUbkNwenh3TCtSdno0RjlheksvODJxeTdyL05ZcVlnRENURXhnT0ZFQVpYODkKMzBoU2wvcnlHZ1dldzhRS2xESUp5cVhTdW9jdUZodk5NMVJkMW1uU1dwd0daYjMyNnEvSkt5RFpDaEVvZy9VcQovdGpUcGZ6V2R5RnNvUXIyeEVlZzV5elNkSDU4eEtzSElJZGhvUkg0NjNsQ3lNNWVxcWROdFNPYW53PT0KLS0tLS1FTkQgUlNBIFBSSVZBVEUgS0VZLS0tLS0K
```
## 创建Secret
可以使用自定义证书。<br />示例：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: skywalking
  namespace: skywalking
data:
  tls.crt: |
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZzekNDQTVzQ0NRQ3h5Q2xkRVVFd2RUQU5CZ2txaGtpRzl3MEJBUVVGQURDQmtURUxNQWtHQTFVRUJoTUMKUTA0eEVEQU9CZ05WQkFnTUIwSmxhV3BwYm1jeEVEQU9CZ05WQkFjTUIwSmxhV3BwYm1jeER6QU5CZ05WQkFvTQpCbWxNWlcxdmJqRVBNQTBHQTFVRUN3d0dhVXhsYlc5dU1Sc3dHUVlEVlFRRERCSjNkM2N1YVd4bGJXOXVkR1ZqCmFDNWpiMjB4SHpBZEJna3Foa2lHOXcwQkNRRVdFR3hwY0dkQWIzVjBiRzl2YXk1amIyMHdJQmNOTWpBd056RTMKTURFMU16VTFXaGdQTWpFeU1EQTNNVGN3TVRVek5UVmFNSUdpTVFzd0NRWURWUVFHRXdKRFRqRVFNQTRHQTFVRQpDQXdIUW1WcGFtbHVaekVRTUE0R0ExVUVCd3dIUW1WcGFtbHVaekVQTUEwR0ExVUVDZ3dHYVV4bGJXOXVNUTh3CkRRWURWUVFMREFacFRHVnRiMjR4TERBcUJnTlZCQU1NSTNOcmVYZGhiR3RwYm1jdGFXNXFaV04wYVc5dUxuTnIKZVhkaGJHdHBibWN1YzNaak1SOHdIUVlKS29aSWh2Y05BUWtCRmhCc2FYQm5RRzkxZEd4dmIyc3VZMjl0TUlJQwpJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBZzhBTUlJQ0NnS0NBZ0VBeCt1V0NRaDJWRTZtYVF5N0FFa0IxdnU1ClV1blJDWjlNcUpHY284c0lkQ2plOGN4WE5QbXdyMXVzOFEyVVAvcUZYWkw5VUJ6R2gyV3FaSTdRc25Dbm5YcVYKcUpyS0IrR3kvTncvWnFEV1hJQnhSYTBsZEdicEkxbjdVSzVyZWJENTJjVGhMeVBQcWtBV2I3ejJRYnFEbTRheQpsV3FwN0h2MThDUHlidklJMUc2MHorUFR1WTdWc2hwTEJhQ0RXdy9pOUdXTCtBWnpSYjdlcnBJeWYxdGtaNjY3CmhlM21aUDc0cklHaVJsK1NHT3BxNnVYQUYwWXRsTFdPMHBCY2ZlaUxlTm9PcUVXUmRqMDVHUkg0eHE2bzZ2YzQKZlNlUzRvYy94R0wwTnVYV0ZENitaZEE2V0tJYm8rQ2wyd21abzFQRXVFcTRxUkJhQXpLZm1EUThrRXpEc28zeQoxb2VwUUVyVFVBUFFqYVZrOThjcU9ROEtIbWEyaUZLOTBoc0IvS0NHNXlpTUF3V0JhckdRTzhuQm5yejZFeEN3CjM2ZlZ4eVpmazloY3R3TzZwWTNibWJuM0ExaVMwNERzNUxhdFZaM3QrN2c5VHJ2Tm91anBHdTJLblhrcU5oODIKa21TKzZ4UEZtZ1Z2dUx6NGtpS3oxbDdmaThzVzFQd2RBdHFKQkJsZDY4MHA3ek5yeFp4MTlVNjVobkl5dVh6Wgo1WHZSYzI4RUNhWVFhNTQybS9rVzU0YkN1QUo1TkRmVnVhWGNiZThrN2hlYk1RaGF1Zk4xRy8rTTJMVXJvczdCCi9aRnFYdWtjQ3F1KzczdWRTMkJCM2pWeG9FTTV3ajNZM0ZpYUptSUh4blhOL2VzcFZ5dHZNU3hQVVpjdWlieWwKSGVLRDcrZEsvK2dxcFhsd3Awc0NBd0VBQVRBTkJna3Foa2lHOXcwQkFRVUZBQU9DQWdFQUhzeDcySjVpMjdTZwpqb2g2N2hnbDBpZUt3ajdrVnljM2JWUmZvQS83NTgweUVQMjl2MWMwbksyVktpYzYrcHM0eGt2WHRpKzVsNVdlCmpmb0p1TmFFVVNHN0xTNVBZUm12aUJQZEVXY2ZteWdRYzhTcjdYeFVreVJyaGFuMnFLWGZwZkVMMmh3YVl1N2wKano2OEdjS0dPL3RJTUphWFRZWHVNQm9SeTdZY0JzY2tXejlIcTZjYndkZXhhdEo5eUhjenF5ZzBUUXZrRXBmMQpSck1DWE0yZHp1Wm1neHFmNkhzamp3Zm9KcWJGS3IrZDBBNjlBUGNJT0N3QUlaSU9VNzIxNko1Mms0N3JINnVaCmRGdWR4QUU5UExSWVVsQTQ2S1RKemVETmxrZGxRSnNQU05QclB0MTdGTmpNbTBNN1dLWDh0ZW9BUHNSWDkwZm4Kd203Qlc3Z2RwQVJ0NGhWdWQ2cmxycmlZWUpKQXNZRjNHV1lGT2RuaVJraHVycGRuUmgza1ZrZU53bDhOb3ZmTgpoY2tnQ0Rvb09jMHZpNHNvNXBwMUFSWHlMWUFsUnlaU2NHQ3U5WTJsUkwvTGJOb1JlRGgrcEJNbGtSa2J5S3ZyCkpGYUZxbVIvVFhXVkRwVFNDNGhQcmkzOVE3eDRJVDRUQUpZQjAvM2FTT1Nlczl0YXZSeUhWREFWR3MvSzJxaDAKeEpaK0c5NVZqQ1ZjWTU2NDBuNzZycDNZZEVlMXdteU9weWhCOSthZ0JMZGlxSVMzQ3ppbUJJeEhRc3hsdXF6Zwo1Ky9Ha1crZTh0TDBKRnVKZ2hkSE1wTzZkaW5XMDVCN2xadWlVSisxS2lsTWZZVmQ0K1poTlp6bDRmcGZhazg2CmxmdXg2UzNOSktVcFQzU3dIRW5ZOXBoRmVsLzlHMms9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  tls.key: |
    LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlKSndJQkFBS0NBZ0VBeCt1V0NRaDJWRTZtYVF5N0FFa0IxdnU1VXVuUkNaOU1xSkdjbzhzSWRDamU4Y3hYCk5QbXdyMXVzOFEyVVAvcUZYWkw5VUJ6R2gyV3FaSTdRc25Dbm5YcVZxSnJLQitHeS9Ody9acURXWElCeFJhMGwKZEdicEkxbjdVSzVyZWJENTJjVGhMeVBQcWtBV2I3ejJRYnFEbTRheWxXcXA3SHYxOENQeWJ2SUkxRzYweitQVAp1WTdWc2hwTEJhQ0RXdy9pOUdXTCtBWnpSYjdlcnBJeWYxdGtaNjY3aGUzbVpQNzRySUdpUmwrU0dPcHE2dVhBCkYwWXRsTFdPMHBCY2ZlaUxlTm9PcUVXUmRqMDVHUkg0eHE2bzZ2YzRmU2VTNG9jL3hHTDBOdVhXRkQ2K1pkQTYKV0tJYm8rQ2wyd21abzFQRXVFcTRxUkJhQXpLZm1EUThrRXpEc28zeTFvZXBRRXJUVUFQUWphVms5OGNxT1E4SwpIbWEyaUZLOTBoc0IvS0NHNXlpTUF3V0JhckdRTzhuQm5yejZFeEN3MzZmVnh5WmZrOWhjdHdPNnBZM2JtYm4zCkExaVMwNERzNUxhdFZaM3QrN2c5VHJ2Tm91anBHdTJLblhrcU5oODJrbVMrNnhQRm1nVnZ1THo0a2lLejFsN2YKaThzVzFQd2RBdHFKQkJsZDY4MHA3ek5yeFp4MTlVNjVobkl5dVh6WjVYdlJjMjhFQ2FZUWE1NDJtL2tXNTRiQwp1QUo1TkRmVnVhWGNiZThrN2hlYk1RaGF1Zk4xRy8rTTJMVXJvczdCL1pGcVh1a2NDcXUrNzN1ZFMyQkIzalZ4Cm9FTTV3ajNZM0ZpYUptSUh4blhOL2VzcFZ5dHZNU3hQVVpjdWlieWxIZUtENytkSy8rZ3FwWGx3cDBzQ0F3RUEKQVFLQ0FnQjZqYm5RMmNZb0VuckM1Ukh4cjgrWDZZSUhDTnlqS0d0RzN0U0dPQjE3ajk1em5jOUw0MURqZnJCNgpxTmRialFpYWYxUVgrenZjbm44ZzI4ZWx5NWVFMnpQSlF2SmNlSWhQcC9pWndqa0dSWnFxbXVUZUFZUWpycko0CkdDUThXamdmdmloSXpJd3VFdEZ1T0FsQjY4Tm40Tit5NXNzNTlCZWsyWFJ4ZkovYkVlNTNqbk1wU2gxSDdBUlgKYzdseVpqKzdrRkNJZi9QRUtxaTVuVmVsYVBJWStTMXRnWlArRDZ1RG5JaE0yYU5tOExVUVFEMU1SeklEK25TTwpNZjd4RkZBRGlyRDNpSVlPVktPZThieHVsMzFuWkp2VlpTQVBDOWdVWEpEVzRuZ2t1djNlemhJbWFXSEdTUjJECk05UmlJaVR5MEtONHlGUlZwMy9jUHhGdGFCR0NXc2ZTM0U4endyellCWkxDK3E5aHQ1REhWRDFFZVVja0wyVEsKc1IvOEJRaGtqd3A4UnI1NmJqVURGSHg3REhUQ0xia3RObEdIRS9xZng3bzZLME85UDMrcVJNUFJkVExsTUNsWApGbW03NEFKNnFlVlZMelZZSXBCMzI1UEpmZHFHb3NXWEZGd09IZEZLMWF3ZGU3VTZTcjV1dWkxOHROTTA2dzY2CjJQL2NBRWVveXlPYktjblJRR3RUOEh3M1Roem1VTGxVVnhBV0c5QytKS2YyQU84Z1Rwc3NFeGdCTmRid244UlIKUCtkbHRXdjhzczZkQzNoWld4OHFPbG1id09VRFBDd2pwSytjSUhyT21XS3F3WW1uc1dhLzhtRkxGTVdFbE9CWgprMGwwa0RhcTVrQmVRRDRkT0ozZ1RybVRSVGVGS20wNVk2OHk2Wm4yZEpKZFUyK29vUUtDQVFFQThNM2lPeUhaCk1CM3FFUmROeFBJQmxOdExFbVp1ck1qVkJ2MVFrUG91eTVPdlRlTjI0UnhVZzZ0ZjZONlIraVVJOGFIcDdvRFkKZzYvVW43UUFldGNhZ05sd3dZSTAyL3AyOEZrdlF3NXdCZy9VNmpIOEVNMnFiWGtxUFNhVTQ5KzJpTWxldytkRApqTFNUaDlaSE5tTWpIMU1iRXlUSjlvYU9MZk55NTVzZS9wNUdsRi9ONm02aWlyci9oYTlMakszMlpodDBVQmJLClEvQi9oeTFCUHNRU1ViL2dXczVsNjduQXgrWmxHSmQzMkJnc1pYZ1pJc2JybVdxQWdCQXpEOTcxUDVZL3hZSEcKaEpYV3IzS2kxaVZOMHJiYjNYS3JuWU5XWVpXVDlGc1JveGRkNmlBNGIwcHUxYkRHUDgwS2dqVmtMaHQ4Ung2aQppbzNGb0lUYW9HQS9ld0tDQVFFQTFJazhFdHhlUVB0cEI1TVltajYrK0R4clZ0SDdVbGlEWisrZTliVnEvbVhwCm9KQS9uaTR3ajlaSXRneklxYUFUcUVZQXJrd0NPRno1Z1VBTk5JZG1KZm1YVFJQWE1OOThGL21QS0NHYzIycVkKWVJGM0V5ZThvc01jNFNDRG13dUFIdmVLbUdla1ovU0lESUE3L3FGRjVvT1BBeFlCdFhmYWw1NkY4S2dEQ2pJdApiVjZwaWFXWXRibldHUktrZ1lub2MzUkZLWlhjVitsWm1DclVIVFZZUFZTTklaRTdjeCtnOThuMUtvZDdlcFAwCkF1MWU5VW5iMDJVcWNNWlR4Z0pWWmxlZ20yQjdTTVNIUGhIRG9iYUJnQVMvRDBhV2psL0NHc1IwbjIvVnlQZWoKMkZpeHEwYnBEb1Avb0ZGMFdjNWt5YVk1WXBxR1ZJN2pwQjc2WVB0R2NRS0NBUUJtSUx1SkhyTHhVRTZjVHdqaAp0VTNzWnhFd3paYTAyenhXQ1l0bzRXS0ZYcTVqMTB2czcwbjh5ZXNVWllqOTBVVTR6ZFhjSEhNM2JwUTc0MmlzCnNKd0I5N1oxcTVabGtaVnVIRFVLT0k1VVU5U1dPdnJQMlpvbGNkVHpPSEMvTUh0SENqaVIxM3h6cVlaRWZ5R1cKdEhjSjZ3STVuVW0ycjV5UGFvZmFLSlQ5b2YwSElwYS9qWXFNVWdEYkh0eG8zc0w0elQ0SVZaSVZQR09wM2d5awo1UjFyNUhmdnEwZllzT0JOSjY3S3Zqckt3SDNienhMQ0NDR1Y0SEI5WCtGNElyUmE4b3M3WmUzZS8zbWMwWDYwCldmL21sVDBzYjF6UG9nczV1ZW1WSWlhWngraEJ5c2FFbE1uckFWMGpZLy9IOTRMMGR4Mi9RQVBweFhVQ3ZqWEcKL09zUEFvSUJBQ2w2VnVUL2FNMlp3eGVkVjNIU3lRWTlGRjJ2UVRNQzN6SkxpMlE4Y1pNUVpKWFhVUE1YNUpaTgpwWWNSMjU5dkJDY1U4bEVSdUdVemorWHZrQ1F1UUhzc3lLQmNQT1RtMkpyOU16a0x4a2J6K0tVSStZb2RlU3p5Cmt4SkVmQ3FvQmVzRk9HRWg2TzM0eFF4YmdsbmhNM2tYTjdQTWVhem1wUnh0N1luWDRDLy9OdXBmVFFyMTdHV2EKelBPN3Rhc3FOTmxEV2JRUjhSeWZCOE13cUpEQ3RtUFJ1bkZRbkorTkRxY05IUEdkMnpWQzVwNGpvU0tSdFhyKwpCVmhZL0VScWRDd3k4YjhoQnk1ZVhGNTZyUWxFU3VXbkIzcDl0aWcyKy9TcFhuRTNMQ3U2V0ptQ1Jibzlib0pNCnRIaWZMT3ZSNGMyUXlTWTdMZmdXb3NxRzJTcFFINkVDZ2dFQUpCdERxd0NReU5NdTZwQ3FGelNNc212TnR1cXUKSmlXSEUrNFdQRTBZd3IwWCtzeFdlUWVBdUo2bGNlWTZFd3Urb0ZZYnJZakVQSUNPVEgyVnZRZUNGWXZSTk9uWQpCdE0xcWQzYTVoWVpPdjhTM1ZWVjNlcHNmR2NBSUxQOTJVR1FITlBoTWZuNGdjaXJla0w5V1FWOUFKNkFLN0tBCjVDdTMwcGJyMTQ0d2NiaVBTc2VUbkNwenh3TCtSdno0RjlheksvODJxeTdyL05ZcVlnRENURXhnT0ZFQVpYODkKMzBoU2wvcnlHZ1dldzhRS2xESUp5cVhTdW9jdUZodk5NMVJkMW1uU1dwd0daYjMyNnEvSkt5RFpDaEVvZy9VcQovdGpUcGZ6V2R5RnNvUXIyeEVlZzV5elNkSDU4eEtzSElJZGhvUkg0NjNsQ3lNNWVxcWROdFNPYW53PT0KLS0tLS1FTkQgUlNBIFBSSVZBVEUgS0VZLS0tLS0K
```
## 部署准入控制器 - SWKAC
准入控制器提供大量的环境变量以及参数可配置，清单如下：

| 参数名称(环境变量) | 默认值 | 说明 |
| --- | --- | --- |
| SWKAC_USE_TLS | true | 是否启用HTTPS |
| SWKAC_TLS_CERT | /etc/swkac/tls.crt | HTTPS证书路径 |
| SWKAC_TLS_KEY | /etc/swkac/tls.key | HTTPS证书路径 |
| SWKAC_TRIGGER_ENV | false | 通过POD环境变量配置，来识别是否启用自动注入功能。<br />true：如果目标POD存在环境变量SWKAC_ENABLE且值为true是才会加载agent，否则不对pod做任何变更。<br />false：关闭该检测功能。 |
| SWKAC_SW_IMAGE | lipangeng/skywalking-initcontainer:latest | 默认的InitContainer镜像地址 |
| SWKAC_SW_AGENT_COLLECTOR_BACKEND_SERVICES | skywalking-aop.skywalking:11800 | 默认的Skywalking AOP地址 |
| SWKAC_SW_JAVA_ENV_NAME | JAVA_TOOL_OPTIONS | 写入到pod中的环境变量，将增加启动命令至该环境变量中，通常设置为：-javaagent:/opt/skywalking/skywalking-agent.jar |

示例：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: skywalking-injection
spec:
  replicas: 3
  template:
    metadata:
      name: skywalking-injection
      labels:
        app: skywalking-injection
    spec:
      containers:
        - name: skywalking-injection
          image: lipangeng/skywalking-injection:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
              name: http
            - containerPort: 443
              name: https
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: SWKAC_TRIGGER_ENV
              value: "true"
            - name: SKAC_OPTIONS
              value: -v 3
            - name: SWKAC_SW_IMAGE
              value: lipangeng/skywalking-initcontainer:7.0.0
            - name: SWKAC_SW_AGENT_COLLECTOR_BACKEND_SERVICES
              value: skywalking-aop.skywalking:11800
            - name: SWKAC_SW_JAVA_ENV_NAME
              value: CATALINA_OPTS
          resources:
            requests:
              memory: 64Mi
            limits:
              memory: 128Mi
          livenessProbe:
            httpGet:
              port: https
              path: /health
              scheme: HTTPS
          readinessProbe:
            httpGet:
              port: https
              path: /health
              scheme: HTTPS
          volumeMounts:
            - mountPath: /etc/swkac/
              name: cert
              readOnly: true
      restartPolicy: Always
      volumes:
        - name: cert
          secret:
            secretName: skywalking
  selector:
    matchLabels:
      app: skywalking-injection
```
## 创建Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: skywalking-injection
  namespace: skywalking
spec:
  selector:
    app: skywalking-injection
  ports:
    - port: 80
      name: http
    - port: 443
      name: https
```
## 注册Skywalking Injection准入控制器
> 需要Kubernetes1.9版本以上可用，且以1.16为分界点，两种不同的配置。

本示例中，对于哪些pod会被拦截做了筛选。<br />如果namespace被打上skywalking-injection=true标签，它下面发布的pod才会经过该准入控制器。除此之外还支持其它规则，详情参考官方文档。<br />示例：
```yaml
# >= 1.16
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: skywalking-injector
webhooks:
  - admissionReviewVersions: ["v1beta1"]
    failurePolicy: Fail
    clientConfig:
      caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZvakNDQTRvQ0NRREZYZTRwM1o0NjREQU5CZ2txaGtpRzl3MEJBUXNGQURDQmtURUxNQWtHQTFVRUJoTUMKUTA0eEVEQU9CZ05WQkFnTUIwSmxhV3BwYm1jeEVEQU9CZ05WQkFjTUIwSmxhV3BwYm1jeER6QU5CZ05WQkFvTQpCbWxNWlcxdmJqRVBNQTBHQTFVRUN3d0dhVXhsYlc5dU1Sc3dHUVlEVlFRRERCSjNkM2N1YVd4bGJXOXVkR1ZqCmFDNWpiMjB4SHpBZEJna3Foa2lHOXcwQkNRRVdFR3hwY0dkQWIzVjBiRzl2YXk1amIyMHdJQmNOTWpBd056RTMKTURFME5EQXhXaGdQTWpFeU1EQTNNVGN3TVRRME1ERmFNSUdSTVFzd0NRWURWUVFHRXdKRFRqRVFNQTRHQTFVRQpDQXdIUW1WcGFtbHVaekVRTUE0R0ExVUVCd3dIUW1WcGFtbHVaekVQTUEwR0ExVUVDZ3dHYVV4bGJXOXVNUTh3CkRRWURWUVFMREFacFRHVnRiMjR4R3pBWkJnTlZCQU1NRW5kM2R5NXBiR1Z0YjI1MFpXTm9MbU52YlRFZk1CMEcKQ1NxR1NJYjNEUUVKQVJZUWJHbHdaMEJ2ZFhSc2IyOXJMbU52YlRDQ0FpSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0lQQURDQ0Fnb0NnZ0lCQUxIUS9ndTE0WUJDYjNRNSthTWtyT3dEeUpQKzQvQUpjRTg0Yk5tci9WbFhEMW9JCnMzTDJvclFicGtRQ1VEVmZqVHFOTWtvT3QrbVpYWHdtMnUzMG1TTkpCRmhaeHEzT2dLSVZxV2IxVWQ5WDRBMjMKOUlxSWREV0NOMndVWlhVd0ZuTTdCYW5xUyt3cmJob3ZxalNFcUI2OUJpbkdjamZvTCsxaWUzdVhqeVBHZklWagpieldVSUdnYzNpVEllc2xyL1prUzBLaDYzbFVtN0NaSFBacVpwMy9vVVNYQmszTXN4dkpCdFZ4QlRQNG5FMCtiCmYvWEkveElTby91ckI1a2RYZW1RM2lnRUtRQmpTWXAvbkthdWEvUHJrZ3BHbVl1a2RlWElVeXhKQk9rNUJCdXcKb3Fid0NBcEFpQ0wzeUMvZnVreEdPZjd6dHdiRGZYU2hXT3ZTSks2NFd0Y1owRHhsQ05VTDFOZzZrcjFLMjRNeApHazZjRDViUWJjWCtEVzhTS25iS0NabzhLR1Jvdjl1cG9tRVc0Q2kzWE8rTjF6cGRST2lwYVpXVTNLdnJGZ2E2CjY5ZmhJM0dnM2RVbFprK2hUazVkMUExNysxU3l3QXRRaldxeG93R3ljY1pPSnRqOUpZaUlITXJsRU9SeWZQTzIKNVg0UG1VSmtoSW9BYWNSdm1oaWRTRFlRTmNMZUt4Ym1TVXdWZEhaSkpGWjJaVjFpcnZPMmM4VEtqNlBXZDA3YQpGNTRvK0JLMVQyMjFWK0hnUnJWbTlaNE5zQWNNc1BScHYzYWNCdk1aWWxxL0h4amttU1NNSHFtd0UwY1d1c28yCkZxdmNiMXBKUGJMdGpQZ1d3VmRPblZxTUc3aTEzYU5uck1mODVKc2VzZVp6VE9zLzFCdlhyODUxVTVMZkFnTUIKQUFFd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dJQkFJWjdaWFpXNEhKMTI3enh2T1ltSHU3aC9rNUdwb294dkorcApZS29QRjRpd3BGRGM3ZjBQZHk0K0tVcG1RS2xjYVJuSEZWWGQ5WEo0R1lnNjg5a0xUeVo3c29iam5RdDZOU0tOCmxYTmdvSUp3Zm1ONXpqV25HUCtXTWhvQlJMM0o0bjNTWUdtalkwZTRzOU1wUlBRdklTeUdZcytFQUQxVGozZEQKdnFoM05WTnd2bjJsNkkyY0lxUEVoU01SY3QwU3lPc2o4MDIveFBvajZtZStqRjk4ZGc2V1M5VUhBSnNqcUV6WApzbHdhQTl6c3ZwQlo0TjVjdXNRMmpHWEdEbkVDQUtMNm1idnRVWHBYd012SFd5MG1vQ1g4c0tKcEJkNlZiUHhMCnhlT1duTnpDRWQvYXdqZVk1N3BzOTNidjZVejZ1a2UyV2ZiOUdMd1BCRmQ5RUs1NkV5Y2U5d2daSThvaDJOSVkKYjY2RHAwbExhR2k0OGRZWkpGRU1ETFhOVUV2RXRmZEo3K3lsU1cyWG52bkxDT25oQjJiQm93YmxlMDc1UllzagpsdFVKRWw1YTFZbUhOWnMzeGdyd1ErL3N4bjg1YlArempTSUlqUms1V1Bpbm4vL25vdGp6UHJJSGJmTjY0by95CjAvVWlrZVRjVFZuZUREQUJwdGZ5VXlEbU5pVlhCaHlodks3MTJEZmZ1Q01aclQ1YlF6VUd1cytBUGo4QThoZ0kKbXFSd25LdnJvcHJ2OEp3b2VZbm1rNExkZmxMSlFYczJVdWd2L1p1VmVkeTFBSk5rZThUSXhhRUoyS2hTMGdacQowWXZEVkFRRUg0VlRMeDhWRTdxRm42QTVhaERtVUF0ZEJ5MUNkSy9JQ1o4eDdKZkRCYS9zK0p1K1ZBTGt5Mm9hCjRxVlNEUHQyCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
      service:
        name: skywalking-injection
        namespace: skywalking
    namespaceSelector:
      matchLabels:
        skywalking-injection: "true"
    name: injection.skywalking.ilemontech.com
    timeoutSeconds: 5
    sideEffects: None
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations:  ["CREATE"]
        resources:   ["pods"]
        scope: "Namespaced"

# > 1.9 & < 1.16
---
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: skywalking-injector
webhooks:
  - failurePolicy: Fail
    clientConfig:
      caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZvakNDQTRvQ0NRREZYZTRwM1o0NjREQU5CZ2txaGtpRzl3MEJBUXNGQURDQmtURUxNQWtHQTFVRUJoTUMKUTA0eEVEQU9CZ05WQkFnTUIwSmxhV3BwYm1jeEVEQU9CZ05WQkFjTUIwSmxhV3BwYm1jeER6QU5CZ05WQkFvTQpCbWxNWlcxdmJqRVBNQTBHQTFVRUN3d0dhVXhsYlc5dU1Sc3dHUVlEVlFRRERCSjNkM2N1YVd4bGJXOXVkR1ZqCmFDNWpiMjB4SHpBZEJna3Foa2lHOXcwQkNRRVdFR3hwY0dkQWIzVjBiRzl2YXk1amIyMHdJQmNOTWpBd056RTMKTURFME5EQXhXaGdQTWpFeU1EQTNNVGN3TVRRME1ERmFNSUdSTVFzd0NRWURWUVFHRXdKRFRqRVFNQTRHQTFVRQpDQXdIUW1WcGFtbHVaekVRTUE0R0ExVUVCd3dIUW1WcGFtbHVaekVQTUEwR0ExVUVDZ3dHYVV4bGJXOXVNUTh3CkRRWURWUVFMREFacFRHVnRiMjR4R3pBWkJnTlZCQU1NRW5kM2R5NXBiR1Z0YjI1MFpXTm9MbU52YlRFZk1CMEcKQ1NxR1NJYjNEUUVKQVJZUWJHbHdaMEJ2ZFhSc2IyOXJMbU52YlRDQ0FpSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0lQQURDQ0Fnb0NnZ0lCQUxIUS9ndTE0WUJDYjNRNSthTWtyT3dEeUpQKzQvQUpjRTg0Yk5tci9WbFhEMW9JCnMzTDJvclFicGtRQ1VEVmZqVHFOTWtvT3QrbVpYWHdtMnUzMG1TTkpCRmhaeHEzT2dLSVZxV2IxVWQ5WDRBMjMKOUlxSWREV0NOMndVWlhVd0ZuTTdCYW5xUyt3cmJob3ZxalNFcUI2OUJpbkdjamZvTCsxaWUzdVhqeVBHZklWagpieldVSUdnYzNpVEllc2xyL1prUzBLaDYzbFVtN0NaSFBacVpwMy9vVVNYQmszTXN4dkpCdFZ4QlRQNG5FMCtiCmYvWEkveElTby91ckI1a2RYZW1RM2lnRUtRQmpTWXAvbkthdWEvUHJrZ3BHbVl1a2RlWElVeXhKQk9rNUJCdXcKb3Fid0NBcEFpQ0wzeUMvZnVreEdPZjd6dHdiRGZYU2hXT3ZTSks2NFd0Y1owRHhsQ05VTDFOZzZrcjFLMjRNeApHazZjRDViUWJjWCtEVzhTS25iS0NabzhLR1Jvdjl1cG9tRVc0Q2kzWE8rTjF6cGRST2lwYVpXVTNLdnJGZ2E2CjY5ZmhJM0dnM2RVbFprK2hUazVkMUExNysxU3l3QXRRaldxeG93R3ljY1pPSnRqOUpZaUlITXJsRU9SeWZQTzIKNVg0UG1VSmtoSW9BYWNSdm1oaWRTRFlRTmNMZUt4Ym1TVXdWZEhaSkpGWjJaVjFpcnZPMmM4VEtqNlBXZDA3YQpGNTRvK0JLMVQyMjFWK0hnUnJWbTlaNE5zQWNNc1BScHYzYWNCdk1aWWxxL0h4amttU1NNSHFtd0UwY1d1c28yCkZxdmNiMXBKUGJMdGpQZ1d3VmRPblZxTUc3aTEzYU5uck1mODVKc2VzZVp6VE9zLzFCdlhyODUxVTVMZkFnTUIKQUFFd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dJQkFJWjdaWFpXNEhKMTI3enh2T1ltSHU3aC9rNUdwb294dkorcApZS29QRjRpd3BGRGM3ZjBQZHk0K0tVcG1RS2xjYVJuSEZWWGQ5WEo0R1lnNjg5a0xUeVo3c29iam5RdDZOU0tOCmxYTmdvSUp3Zm1ONXpqV25HUCtXTWhvQlJMM0o0bjNTWUdtalkwZTRzOU1wUlBRdklTeUdZcytFQUQxVGozZEQKdnFoM05WTnd2bjJsNkkyY0lxUEVoU01SY3QwU3lPc2o4MDIveFBvajZtZStqRjk4ZGc2V1M5VUhBSnNqcUV6WApzbHdhQTl6c3ZwQlo0TjVjdXNRMmpHWEdEbkVDQUtMNm1idnRVWHBYd012SFd5MG1vQ1g4c0tKcEJkNlZiUHhMCnhlT1duTnpDRWQvYXdqZVk1N3BzOTNidjZVejZ1a2UyV2ZiOUdMd1BCRmQ5RUs1NkV5Y2U5d2daSThvaDJOSVkKYjY2RHAwbExhR2k0OGRZWkpGRU1ETFhOVUV2RXRmZEo3K3lsU1cyWG52bkxDT25oQjJiQm93YmxlMDc1UllzagpsdFVKRWw1YTFZbUhOWnMzeGdyd1ErL3N4bjg1YlArempTSUlqUms1V1Bpbm4vL25vdGp6UHJJSGJmTjY0by95CjAvVWlrZVRjVFZuZUREQUJwdGZ5VXlEbU5pVlhCaHlodks3MTJEZmZ1Q01aclQ1YlF6VUd1cytBUGo4QThoZ0kKbXFSd25LdnJvcHJ2OEp3b2VZbm1rNExkZmxMSlFYczJVdWd2L1p1VmVkeTFBSk5rZThUSXhhRUoyS2hTMGdacQowWXZEVkFRRUg0VlRMeDhWRTdxRm42QTVhaERtVUF0ZEJ5MUNkSy9JQ1o4eDdKZkRCYS9zK0p1K1ZBTGt5Mm9hCjRxVlNEUHQyCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
      service:
        name: skywalking-injection
        namespace: skywalking
    namespaceSelector:
      matchLabels:
        skywalking-injection: "true"
    name: injection.skywalking.ilemontech.com
    sideEffects: None
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations:  ["CREATE"]
        resources:   ["pods"]
```