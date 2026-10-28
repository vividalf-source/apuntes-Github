# apuntes-Github

# Apuntes de Terraform – Evaluación 2

Resumen corto de: `count` / `for_each`, variables y outputs, módulos, `locals` + `random`, provisioners y debugging.

---

## 0. Comandos básicos

```bash
terraform init          # descarga providers y módulos (siempre primero)
terraform validate      # revisa la sintaxis
terraform plan          # muestra qué va a cambiar
terraform apply         # aplica los cambios
terraform destroy       # elimina todo
terraform output        # muestra los outputs
terraform state list    # lista los recursos creados
```

En el plan: `+` crea · `-` destruye · `~` modifica · `-/+` recrea.

---

## 1. `count` y `for_each`

Crean **varios recursos** con un solo bloque.

```hcl
# count: recibe un número, se identifica por índice
resource "aws_instance" "srv" {
  count         = 3
  ami           = var.ami
  instance_type = "t2.micro"
  tags          = { Name = "srv-${count.index}" }   # 0, 1, 2
}
# Acceso: aws_instance.srv[0].id   |   aws_instance.srv[*].id (todos)

# for_each: recibe un map o set, se identifica por llave
resource "aws_instance" "srv" {
  for_each      = toset(["web", "api"])             # lista -> toset()
  ami           = var.ami
  instance_type = "t2.micro"
  tags          = { Name = each.key }
}
# Acceso: aws_instance.srv["web"].id
```

| | `count` | `for_each` |
|---|---|---|
| Recibe | número | map o set |
| Identifica por | índice `[0]` | llave `["web"]` |
| Variables | `count.index` | `each.key` / `each.value` |
| Si borras uno del medio | se corren los índices y se recrean recursos | solo se borra ese |

No se pueden usar los dos en el mismo bloque.

---

## 2. Variables y Outputs

```hcl
# variables.tf
variable "region" {
  type    = string
  default = "us-east-1"      # sin default, Terraform lo pide
}
variable "subnets" {
  type = list(string)
}

# outputs.tf
output "vpc_id" {
  description = "ID de la VPC"
  value       = aws_vpc.this.id
}
```

Uso: `var.region` · `var.subnets[0]` · `terraform output vpc_id`

Valores por fuera: `terraform.tfvars`, `-var="region=us-west-2"` o `-var-file="x.tfvars"`.

---

## 3. Módulos

Un módulo es **una carpeta con archivos `.tf`**. El módulo hijo solo recibe lo que el raíz le pasa y solo entrega lo que exporta con `output`.

```
proyecto/
├── main.tf  variables.tf  outputs.tf     # raíz
└── modulos/
    └── ec2/
        ├── main.tf  variables.tf  outputs.tf
```

```hcl
# main.tf (raíz): llamar al módulo
module "web" {
  source    = "./modulos/ec2"              # ruta local
  subnet_id = module.vpc.subnet_ids[0]     # output de otro módulo
}

# usar su output:  module.web.instance_id
```

Módulo del registry (acá sí va `version`):

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

Después de agregar o cambiar un módulo → `terraform init`.

---

## 4. Locals y Random

```hcl
resource "random_string" "suf" {
  length  = 6
  special = false
  upper   = false
}

locals {
  prefijo = "demo-${random_string.suf.result}"
  tags    = { Proyecto = "evaluacion" }
}

# Uso: local.prefijo  (sin "s" al usarlo)
resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"
  tags       = merge(local.tags, { Name = "${local.prefijo}-vpc" })
}
```

- `locals` = valores calculados **adentro** · `variable` = valores que **entran**.
- `random_string` se guarda en el estado, no cambia en cada apply.

---

## 5. Provisioners

Corren cuando se **crea** el recurso.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = "t2.micro"
  key_name      = var.key_name

  provisioner "local-exec" {                 # en MI computador
    command = "echo ${self.public_ip} >> ips.txt"
  }

  connection {                               # solo para remote-exec
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("llave.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {                # DENTRO del servidor
    inline = ["sudo apt-get update -y", "sudo apt-get install -y nginx"]
  }
}
```

| | `local-exec` | `remote-exec` |
|---|---|---|
| Dónde corre | mi máquina | el servidor |
| `connection` | no | sí (SSH) |

Si un provisioner falla, el recurso queda **tainted** y se recrea en el próximo apply.

---

## 6. Debugging y Taint

```bash
export TF_LOG=DEBUG                 # TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG_PATH=./terraform.log  # guarda los logs en un archivo
terraform plan
cat terraform.log
unset TF_LOG TF_LOG_PATH            # desactivar
```

Se necesitan **las dos** variables para guardar el log en archivo.

```bash
terraform taint aws_instance.web                 # marca para recrear
terraform apply -replace="aws_instance.web"      # forma moderna (taint está deprecado)
terraform untaint aws_instance.web               # quitar la marca
```

---

## 7. Errores comunes

| Error | Solución |
|---|---|
| Falta `terraform init` tras agregar módulo o provider | Ejecutar `terraform init` |
| `for_each` con una lista | `toset(var.lista)` |
| `aws_instance.srv.id` con `count` | `aws_instance.srv[0].id` |
| `random_string.suf` sin `.result` | `random_string.suf.result` |
| Usar un recurso interno de un módulo | El módulo debe exportarlo con `output` |
| `version` en un módulo local | `version` es solo para módulos del registry |
| `host = aws_instance.x.public_ip` dentro de su `connection` | `self.public_ip` |

Para subir a GitHub **no incluyas**: `*.tfstate`, `.terraform/` ni credenciales/llaves `.pem`.
