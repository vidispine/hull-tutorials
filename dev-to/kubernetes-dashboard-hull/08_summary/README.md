
# Summary

To finish this tutorial let's do some comparisons. For this copy over once more the last chart state which is also the final one and make the `values.full.yaml` the `values.yaml`:

```sh
cd ~/kubernetes-dashboard-hull && cp -R 07_bits_and_pieces/ 08_summary && cd 08_summary/kubernetes-dashboard-hull && rm values.yaml && cp values.full.yaml values.yaml
```

## Lines of Code

Assuming you have created that functionally resembles the original `kubernetes-dashboard` Helm chart, let's consider some numbers. 

Here is a Lines of Code analysis performed with the Visual Studio Code LoC Extension for the original `kubernetes-dashboard` charts relevant configuration files (`values.yaml` + templates):

| File | Lines of Code | Comments
| ------------------------------- | ----------------------------------------------------------------| -----------------------------
| `values.yaml` | 149 | 161
| `templates/_helpers.tpl` | 64 | 15
| `templates/_tplvalues.tpl` | 23 | 5
| `templates/clusterrole-metrics.yaml` | 33 | 1
| `templates/clusterrole-readonly.yaml` | 153 | 1
| `templates/clusterrolebinding-metrics.yaml` | 36 | 1
| `templates/clusterrolebinding-readonly.yaml` | 36 | 1
| `templates/configmap.yaml` | 33 | 1
| `templates/deployment.yaml` | 186 | 2
| `templates/ingress.yaml` | 88 | 1
| `templates/networkpolicy.yaml` | 44 | 1
| `templates/pdb.yaml` | 38 | 1
| `templates/psp.yaml` | 81 | 1
| `templates/role.yaml` | 48 | 1
| `templates/rolebinding.yaml` | 36 | 1
| `templates/secret.yaml` | 46 | 1
| `templates/service.yaml` | 58 | 1
| `templates/serviceaccount.yaml` | 28 | 1
| `templates/servicemonitor.yaml` | 40 | 1
| Total | 1220 | 198

Essentially there exist ~1220 lines of configurational code written which need to be maintained by the chart maintainers. The chart consumers need to understand and maintain ~150 lines of configuration in the `values.yaml` but will likely need to check the templates too to understand the inner workings.

Compared to that, the `kubernetes-dashboard-hull`'s `values.yaml` - containing all configuration code - consists of a mere 460 lines of configuration. With only about a third of the original configuration code to maintain, the HULL based chart arguably covers the default structure and logic of the source chart. But still it is relatively unrestricted in the ways you can adapt the configuration to your needs in any given deployment scenario.



