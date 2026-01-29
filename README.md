# permissoes-tabelas

Permissões lakeformation:

Perfeito 👍
Abaixo está o conteúdo completo de um arquivo .md explicando como aplicar permissões no Lake Formation com Terraform, incluindo control_account_id (catalog_id) e múltiplas tabelas.

Você pode salvar exatamente como, por exemplo:

lakeformation-multi-table-permissions.md


⸻


# Lake Formation – Permissões em Múltiplas Tabelas com Terraform

Este documento mostra como conceder permissões no **AWS Lake Formation** usando **Terraform**, incluindo:
- Múltiplas tabelas
- Permissões `DESCRIBE`, `SELECT`, `DELETE`, `ALTER`
- Uso de **`control_account_id` para especificar o catálogo (cross-account ou centralizado)**

---

## Conceito Importante: `catalog_id`

No Terraform, o Lake Formation **não usa `control_account_id` diretamente**.  
O identificador do catálogo é passado via:

catalog_id

Normalmente:
- Conta local → `catalog_id = data.aws_caller_identity.current.account_id`
- Conta central (control account) → `catalog_id = var.control_account_id`

---

## Variáveis

```hcl
variable "database_name" {
  description = "Nome do database no Glue / Lake Formation"
  type        = string
}

variable "tables" {
  description = "Lista de tabelas que receberão permissões"
  type        = list(string)
}

variable "iam_role_arn" {
  description = "IAM Role que receberá as permissões"
  type        = string
}

variable "control_account_id" {
  description = "Account ID do catálogo (control / governance account)"
  type        = string
}


⸻

Permissões em Múltiplas Tabelas (DESCRIBE, SELECT, DELETE, ALTER)

Usando for_each + catalog_id

resource "aws_lakeformation_permissions" "table_permissions" {
  for_each = toset(var.tables)

  principal = var.iam_role_arn

  permissions = [
    "DESCRIBE",
    "SELECT",
    "DELETE",
    "ALTER"
  ]

  table {
    catalog_id    = var.control_account_id
    database_name = var.database_name
    name          = each.value
  }
}

Esse padrão:
	•	Cria uma permissão por tabela
	•	Funciona para ambientes multi-account
	•	É o modelo mais seguro e controlável

⸻

Boa Prática: Separar Permissões de Leitura e Escrita

O Lake Formation pode falhar quando permissões são misturadas.
Separar evita erros silenciosos.

Permissões de Leitura

resource "aws_lakeformation_permissions" "read_permissions" {
  for_each = toset(var.tables)

  principal = var.iam_role_arn

  permissions = [
    "DESCRIBE",
    "SELECT"
  ]

  table {
    catalog_id    = var.control_account_id
    database_name = var.database_name
    name          = each.value
  }
}

Permissões de Escrita

resource "aws_lakeformation_permissions" "write_permissions" {
  for_each = toset(var.tables)

  principal = var.iam_role_arn

  permissions = [
    "DELETE",
    "ALTER"
  ]

  table {
    catalog_id    = var.control_account_id
    database_name = var.database_name
    name          = each.value
  }
}


⸻

Permissões para TODAS as Tabelas do Database (Wildcard)

⚠️ Aplica para tabelas atuais e futuras

resource "aws_lakeformation_permissions" "all_tables_permissions" {
  principal = var.iam_role_arn

  permissions = [
    "DESCRIBE",
    "SELECT"
  ]

  table {
    catalog_id    = var.control_account_id
    database_name = var.database_name
    wildcard      = true
  }
}


⸻

Pré-Requisitos Obrigatórios

1. Terraform Runner como Data Lake Admin

O role que executa o Terraform PRECISA estar em:

Lake Formation → Data lake settings → Data lake administrators


⸻

2. Permissões no Glue

Mesmo usando Lake Formation, o IAM Role precisa de:

glue:GetDatabase
glue:GetTable
glue:GetTables


⸻

3. Tabelas Registradas no Lake Formation

As tabelas devem:
	•	Existir no Glue
	•	Estar registradas no Lake Formation
	•	Apontar para locations válidas no S3

⸻

Resumo

Item	Valor
Multi-account	catalog_id = control_account_id
Múltiplas tabelas	for_each
Evitar erros	separar read / write
Controle total	tabela por tabela
Acesso amplo	wildcard


⸻

Referência
	•	Terraform Provider AWS – Lake Formation
	•	AWS Lake Formation Permissions Model

---

Se quiser, eu posso:
- Converter isso em **módulo Terraform**
