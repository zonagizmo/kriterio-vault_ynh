Servidor central de sincronización para [Kriterio Vault](https://github.com/zonagizmo/kriterio-vault), una aplicación de gestión empresarial y contabilidad que corre normalmente en local (SQLite, sin conexión permanente).

Esta instancia YunoHost actúa como copia central: las instalaciones locales le empujan periódicamente sus cambios (facturas, bancos, contabilidad...) para tener respaldo centralizado y, más adelante, sincronización entre varios PCs/ubicaciones.

No expone interfaz web propia — es solo la API de sincronización, protegida por el SSO de YunoHost.
