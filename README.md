# SQL Server - Documentação Completa 📚

[![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)](https://www.microsoft.com/sql-server)
[![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)](https://github.com/danibaviera/sql_server_documentation)

## 🎯 Sobre

Este repositório contém uma **documentação abrangente e prática** sobre administração, otimização e monitoramento do Microsoft SQL Server. Foi desenvolvido para DBAs, desenvolvedores e profissionais que trabalham com SQL Server no dia a dia.

## 📋 Conteúdo

### 🔐 [Gerenciamento de Acessos](SQL_Server_Documentacao_Completa.md#gerenciamento-de-acessos)
- Autenticação Windows vs SQL Server
- Roles e permissões
- Scripts práticos para criação de usuários
- Auditoria de acessos

### 💾 [Backup e Restore](SQL_Server_Documentacao_Completa.md#backup-e-restore)
- Estratégias de backup (Full, Differential, Log)
- Modelos de recuperação
- Automação com SQL Agent
- Procedures de restore point-in-time

### 🤖 [SQL Server Agent](SQL_Server_Documentacao_Completa.md#sql-server-agent)
- Jobs, Schedules, Alerts e Operators
- Automação de tarefas de manutenção
- Monitoramento e troubleshooting
- Exemplos práticos de implementação

### ⚡ [Tuning e Otimização](SQL_Server_Documentacao_Completa.md#tuning-e-otimização)
- Identificação de gargalos de performance
- Análise de consultas custosas
- Otimização de índices
- Melhores práticas de tuning

### 📊 [Estatísticas](SQL_Server_Documentacao_Completa.md#estatísticas-no-sql-server)
- Importância para o otimizador
- Configuração e manutenção
- Jobs automatizados de atualização
- Scripts de monitoramento

### 🔍 [Seek vs Scan](SQL_Server_Documentacao_Completa.md#seek-vs-scan)
- Diferenças de performance
- Como identificar problemas
- Técnicas de otimização
- Conversão de Scan para Seek

### 📈 [Query Store](SQL_Server_Documentacao_Completa.md#query-store)
- Configuração e habilitação
- Análise de regressões de performance
- Forçamento de planos de execução
- Manutenção automatizada

### 🚨 [Sistema de Alertas](SQL_Server_Documentacao_Completa.md#sistema-de-alertas)
- Alertas para processos bloqueados
- Monitoramento de status dos databases
- Notificações de falhas em jobs
- Alertas de severidade e corrupção
- Configuração do Database Mail

## 🛠️ Recursos Incluídos

- **90+ scripts SQL** prontos para produção
- **Jobs automatizados** para manutenção
- **Consultas de monitoramento** em tempo real
- **Dashboard de performance**
- **Procedures de limpeza** automatizada
- **Configurações otimizadas**

## 🚀 Como Usar

1. **Clone o repositório:**
   ```bash
   git clone https://github.com/danibaviera/sql_server_documentation.git
   ```

2. **Acesse a documentação:**
   - Abra o arquivo [SQL_Server_Documentacao_Completa.md](SQL_Server_Documentacao_Completa.md)
   - Navegue pelos tópicos usando o índice
   - Copie e adapte os scripts para seu ambiente

3. **Implemente gradualmente:**
   - Comece pelos alertas básicos
   - Configure backup automatizado
   - Implemente monitoramento de performance
   - Ajuste conforme necessidades específicas

## ⚠️ Importante

- **Teste sempre** em ambiente de desenvolvimento primeiro
- **Adapte os scripts** para suas necessidades específicas
- **Configure adequadamente** Database Mail e operadores
- **Monitore o impacto** das mudanças implementadas

## 🤝 Contribuições

Contribuições são bem-vindas! Se você tem sugestões, correções ou melhorias:

1. Faça um fork do projeto
2. Crie uma branch para sua feature (`git checkout -b feature/nova-funcionalidade`)
3. Commit suas mudanças (`git commit -am 'Adiciona nova funcionalidade'`)
4. Push para a branch (`git push origin feature/nova-funcionalidade`)
5. Abra um Pull Request

## 📞 Contato

Para dúvidas, sugestões ou discussões sobre SQL Server, sinta-se à vontade para abrir uma issue ou entrar em contato.

## 📄 Licença

Este projeto está sob a licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.

---

⭐ **Gostou do projeto? Deixe uma star!** ⭐

📚 **Encontrou útil? Compartilhe com outros profissionais!** 📚

---

*Desenvolvido com ❤️ para a comunidade SQL Server*