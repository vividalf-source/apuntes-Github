# apuntes-Github

# 📘 Apuntes Evaluación 2 – Infraestructura como Código I (Terraform)

> Temas: funciones repetitivas · variables y outputs · módulos · debugging · locals · remote-exec / local-exec

---

## 1. Comandos básicos

| Comando | Para qué sirve |
|---|---|
| `terraform init` | Descarga providers y módulos, configura el backend |
| `terraform validate` | Revisa la sintaxis |
| `terraform plan` | Muestra qué va a cambiar (no aplica nada) |
| `terraform apply` | Aplica los cambios (`-auto-approve` salta la confirmación) |
| `terraform destroy` | Elimina todo lo que gestiona Terraform |
| `terraform state list` | Lista los recursos del estado (con índices) |
| `terraform output` | Muestra los outputs |
| `terraform providers` / `terraform version` | Providers requeridos/instalados |
| `terraform fmt` | Formatea el código |

Símbolos del plan: `+` crear · `-` destruir · `-/+` destruir y recrear · `~` modificar.

---

## 2. Providers y bloque `terraform`

```hcl
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws    = { source = "hashicorp/aws",    version = "~> 5.0" }
    random = { source = "hashicorp/random", version = "3.1.0" }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

- Un **provider** es un plugin que habla con una API (AWS, Azure, GCP...).
- Sin bloques `resource`, Terraform no crea nada.

---

## 3. Resource, data, variables, locals, outputs

### Resource → crea infraestructura
```hcl
resource "aws_vpc" "vpc" {          # resource "TIPO" "NOMBRE_LOCAL"
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "demo" }
}
# Se referencia así: aws_vpc.vpc.id
```

### Data source → consulta algo que YA existe
```hcl
data "aws_region" "current" {}
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
# Se referencia así: data.aws_ami.ubuntu.id
```

### Variables → entradas
```hcl
variable "vpc_cidr" {
  type        = string
  description = "CIDR de la VPC"
  default     = "10.0.0.0/16"     # opcional (si no hay default, es obligatoria)
}
# Uso: var.vpc_cidr
```
Formas de pasar valores (de menor a mayor prioridad): `default` → variable de entorno `TF_VAR_x` → `terraform.tfvars` → `*.auto.tfvars` → `-var` / `-var-file` (la última gana).

Tipos: `string`, `number`, `bool`, `list(string)`, `map(string)`, `set`, `object({...})`.

### Locals → valores calculados internos (no se pasan desde afuera)
```hcl
locals {
  application = "corp_api"
  server_name = "ec2-${var.environment}-${local.application}"
}
# Uso: local.server_name   (OJO: es "local", sin "s", al usarlo)
```

### Outputs → datos que expone un módulo/proyecto
```hcl
output "vpc_id" {
  description = "ID de la VPC"
  value       = aws_vpc.vpc.id
  sensitive   = false
}
```
Ver con `terraform output` o `terraform output vpc_id`.

### Provider `random`
```hcl
resource "random_string" "suffix" {
  length  = 6
  special = false
  upper   = false
}
# Uso: random_string.suffix.result
```

---

## 4. Funciones repetitivas: `count` vs `for_each`

### `count` (lista indexada: `[0]`, `[1]`...)
```hcl
resource "local_sensitive_file" "users" {
  count    = length(var.users)
  filename = var.users[count.index]
  content  = var.content
}
```

### `for_each` (cada elemento tiene una **clave**)
```hcl
resource "local_sensitive_file" "users" {
  for_each = toset(var.users)      # for_each NO acepta list: usar toset() o un map
  filename = each.value
  content  = var.content
}
```

| | `count` | `for_each` |
|---|---|---|
| Acceso | `count.index` | `each.key` / `each.value` |
| Entrada | número (o `length(lista)`) | `set` o `map` |
| Dirección en el estado | `recurso.x[0]` | `recurso.x["clave"]` |
| Si borras un elemento del medio | **se corren los índices → recrea recursos** | solo afecta a ese elemento ✅ |

**Caso del laboratorio** (`users = ["./user10","./user11","./user12","./user10"]`):
- Con `count`: crea **4** recursos (aunque `./user10` se repite).
- Con `for_each` + `toset()`: crea **3** (el set elimina duplicados).
- Si quitas `./user11` con `count`: los índices se desplazan → 1 destruido + 2 recreados (`-/+`). Con `for_each`: solo se destruye 1.

> Regla: si los elementos pueden cambiar de orden o borrarse, usa **`for_each`**.

---

## 5. Provisioners: `local-exec` y `remote-exec`

```hcl
resource "aws_instance" "web_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
  key_name      = aws_key_pair.generated.key_name

  connection {                      # necesario para remote-exec
    type        = "ssh"
    user        = "ubuntu"
    private_key = tls_private_key.generated.private_key_pem
    host        = self.public_ip
  }

  provisioner "local-exec" {        # se ejecuta en TU máquina
    command = "chmod 600 ${local_file.private_key_pem.filename}"
  }

  provisioner "remote-exec" {       # se ejecuta DENTRO del servidor (por SSH)
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx",
    ]
  }

  lifecycle {
    ignore_changes = [security_groups]
  }
}
```

| | `local-exec` | `remote-exec` |
|---|---|---|
| Dónde corre | En la máquina donde ejecutas Terraform | En el recurso remoto |
| Necesita `connection` | No | **Sí** (SSH/WinRM) |
| Opciones | `command` | `inline`, `script`, `scripts` |

- Los provisioners corren **solo al crear** el recurso (por defecto). Para volver a ejecutarlos hay que recrearlo.
- Son el último recurso: Terraform no puede planificar lo que hacen.

### Forzar recreación de un recurso
```bash
terraform taint aws_instance.web_server    # (antiguo) lo marca como "tainted"
terraform apply -replace="aws_instance.web_server"   # (moderno, recomendado)
terraform untaint aws_instance.web_server  # quita la marca
```
En el plan aparece: `is tainted, so must be replaced` → `Plan: 1 to add, 0 to change, 1 to destroy`.

---

## 6. Debugging

```bash
export TF_LOG=TRACE                      # niveles: TRACE, DEBUG, INFO, WARN, ERROR (TRACE = el más detallado)
export TF_LOG_PATH="terraform_log.txt"   # guarda los logs en un archivo
terraform init -upgrade                  # ejecuta algo y revisa el archivo
export TF_LOG=""                         # desactiva el logging (también: unset TF_LOG)
```
En Windows PowerShell: `$env:TF_LOG="TRACE"` y `$env:TF_LOG_PATH="terraform_log.txt"`.

Errores típicos:
- `terraform validate` falla → error de sintaxis o argumento mal escrito (ej: `file` en vez de `filename`).
- Falta `terraform init` al agregar un módulo/provider nuevo.
- Referencia a un recurso/output que no existe (`module.vpc.subnet_ids` sin declarar el output).

---

## 7. Módulos

- Un módulo = **cualquier carpeta con archivos `.tf`**. El directorio principal es el **módulo raíz**; los llamados son **módulos hijos**.
- Estructura estándar mínima: `main.tf`, `variables.tf`, `outputs.tf`.
- Un módulo hijo **no ve** las variables del raíz: todo lo que necesita se le pasa como argumento, y lo que devuelve sale por `output`.

```hcl
module "server" {                       # module "NOMBRE"
  source    = "./modulos/ec2"           # ruta local o remota (registry)
  ami       = data.aws_ami.ubuntu.id    # argumentos = variables del módulo hijo
  subnet_id = module.vpc.subnet_ids[0]  # usar el output de otro módulo
}
# Usar un output del módulo: module.server.public_ip
```

**Módulo del registry (con versión):**
```hcl
module "s3-bucket" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "~> 4.0"     # restricción de versión
}
```
Restricciones: `= 1.2.3` exacta · `>= 1.2` mínima · `~> 1.2` permite 1.x · `~> 1.2.3` permite 1.2.x.

Después de agregar un módulo → `terraform init`.

Ventajas: reutilización, orden, consistencia entre entornos, más fácil de mantener y escalar.

---

## 8. Backend (estado)

- **Local**: `terraform.tfstate` en tu equipo (sirve para pruebas).
- **Remoto** (S3 + DynamoDB para bloqueo, GCS...): trabajo en equipo, respaldo y *state locking*.

---

## 9. Ejercicio de práctica: VPC + EC2 + NLB + S3 con módulos

**Estructura**
```
.
├── main.tf          # módulo raíz: locals + llamadas a módulos
├── variables.tf
├── outputs.tf
├── terraform.tf     # providers (aws + random)
└── modulos/
    ├── vpc/  (main.tf, variables.tf, outputs.tf)
    ├── ec2/  (main.tf, variables.tf, outputs.tf)
    ├── nlb/  (main.tf, variables.tf, outputs.tf)
    └── s3/   (main.tf, variables.tf, outputs.tf)
```

**Raíz – `main.tf`**
```hcl
resource "random_string" "suffix" {
  length  = 6
  special = false
  upper   = false
}

locals {
  prefix = "${var.project}-${random_string.suffix.result}"
}

module "vpc" {
  source       = "./modulos/vpc"
  prefix       = local.prefix
  vpc_cidr     = var.vpc_cidr           # 10.0.0.0/16
  subnet_cidrs = var.subnet_cidrs       # ["10.0.1.0/24", "10.0.2.0/24"]
}

module "ec2" {
  source    = "./modulos/ec2"
  for_each  = var.servers                # { web1 = 0, web2 = 1 }
  name      = "${local.prefix}-${each.key}"
  subnet_id = module.vpc.subnet_ids[each.value]
}

module "nlb" {
  source    = "./modulos/nlb"
  for_each  = var.nlbs                   # { nlb1 = 0, nlb2 = 1 }
  name      = "${local.prefix}-${each.key}"
  vpc_id    = module.vpc.vpc_id
  subnet_id = module.vpc.subnet_ids[each.value]
}

module "s3" {
  source      = "./modulos/s3"
  for_each    = toset(var.buckets)       # ["data", "logs"]
  bucket_name = "${local.prefix}-${each.key}"
}
```

**Raíz – `variables.tf`**
```hcl
variable "project"      { type = string, default = "demo" }
variable "vpc_cidr"     { type = string }
variable "subnet_cidrs" { type = list(string) }
variable "servers"      { type = map(number) }
variable "nlbs"         { type = map(number) }
variable "buckets"      { type = list(string) }
```

**Raíz – `outputs.tf`**
```hcl
output "vpc_id"      { value = module.vpc.vpc_id }
output "ec2_ids"     { value = { for k, m in module.ec2 : k => m.instance_id } }
output "nlb_dns"     { value = { for k, m in module.nlb : k => m.dns_name } }
output "bucket_names"{ value = { for k, m in module.s3  : k => m.bucket_name } }
```

**`modulos/vpc/main.tf`**
```hcl
data "aws_availability_zones" "az" {}

locals {
  azs = data.aws_availability_zones.az.names
}

resource "aws_vpc" "this" {
  cidr_block = var.vpc_cidr
  tags       = { Name = "${var.prefix}-vpc" }
}

resource "aws_subnet" "this" {
  count             = length(var.subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  tags              = { Name = "${var.prefix}-subnet-${count.index + 1}" }
}
```
`variables.tf`: `prefix`, `vpc_cidr`, `subnet_cidrs` · `outputs.tf`: `vpc_id = aws_vpc.this.id`, `subnet_ids = aws_subnet.this[*].id`

**`modulos/ec2/main.tf`**
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

locals {
  tags = { Name = var.name, Environment = "Training" }
}

resource "aws_instance" "this" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  tags          = local.tags
}
```
`variables.tf`: `name`, `subnet_id`, `instance_type` (default `t2.micro`) · `outputs.tf`: `instance_id`

**`modulos/nlb/main.tf`**
```hcl
locals {
  lb_name = substr(replace(var.name, "_", "-"), 0, 32)   # el nombre del LB admite máx. 32 caracteres
}

resource "aws_lb" "this" {
  name               = local.lb_name
  load_balancer_type = "network"
  internal           = true
  subnets            = [var.subnet_id]
}
```
`variables.tf`: `name`, `vpc_id`, `subnet_id` · `outputs.tf`: `dns_name = aws_lb.this.dns_name`

**`modulos/s3/main.tf`**
```hcl
locals {
  common_tags = { Terraform = "true" }
}

resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  tags   = local.common_tags
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```
`variables.tf`: `bucket_name` · `outputs.tf`: `bucket_name = aws_s3_bucket.this.bucket`

**`terraform.tfvars`**
```hcl
vpc_cidr     = "10.0.0.0/16"
subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
servers      = { web1 = 0, web2 = 1 }
nlbs         = { nlb1 = 0, nlb2 = 1 }
buckets      = ["data", "logs"]
```

**Mostrar debugging (lo piden en el ejercicio)**
```bash
export TF_LOG=DEBUG
export TF_LOG_PATH="terraform_log.txt"
terraform init && terraform plan
export TF_LOG=""
```

---

## 10. Mini-resumen para el día de la prueba

- **Recurso** = crea · **data** = consulta · **variable** = entra · **local** = se calcula adentro · **output** = sale.
- `count` → índices · `for_each` → claves (necesita `set`/`map`).
- `local-exec` = en mi PC · `remote-exec` = en el servidor (necesita `connection`).
- `TF_LOG` activa logs · `TF_LOG_PATH` los guarda · `TF_LOG=""` los apaga.
- Módulo = carpeta con `.tf` · se llama con `module` + `source` · sus outputs se leen con `module.NOMBRE.OUTPUT`.
- Siempre `terraform init` tras agregar módulos o providers.
