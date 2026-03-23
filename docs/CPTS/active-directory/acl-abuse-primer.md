```markdown
# Access Control List (ACL) Abuse Primer

## ℹ️ Informations

| Field | Value |
|-------|-------|
| 🌐 Platform | HackTheBox Academy |
| 📚 Module | Active Directory Enumeration & Attacks |
| 🔗 Section | [1456](https://academy.hackthebox.com/app/module/143/section/1456) |

***

## 🔑 Теорія

### Типи ACL
| ACL | Опис |
|-----|------|
| `DACL` (Discretionary Access Control List) | Визначає які security principals мають або не мають доступ до об'єкта |
| `SACL` (System Access Control List) | Дозволяє адміністраторам логувати спроби доступу до об'єктів |

### Типи ACE (Access Control Entries)
| ACE | Опис |
|-----|------|
| `Access denied ACE` | Явна заборона доступу до об'єкта |
| `Access allowed ACE` | Явний дозвіл доступу до об'єкта |
| `System audit ACE` | Генерує audit logs при спробі доступу |

### Найважливіші права для зловживання (ACE Abuse)
| ACE | Інструмент | Що дозволяє |
|-----|-----------|-------------|
| `ForceChangePassword` | `Set-DomainUserPassword` | Скинути пароль юзера без знання поточного |
| `GenericWrite` | `Set-DomainObject` | Записати будь-який атрибут об'єкта (наприклад SPN → Kerberoasting) |
| `GenericAll` | `Set-DomainUserPassword` / `Add-DomainGroupMember` | Повний контроль над об'єктом → targeted Kerberoasting |
| `WriteOwner` | `Set-DomainObjectOwner` | Змінити власника об'єкта |
| `WriteDACL` | `Add-DomainObjectACL` | Змінити права доступу до об'єкта |
| `AllExtendedRights` | `Set-DomainUserPassword` / `Add-DomainGroupMember` | Всі розширені права |
| `AddSelf` | `Add-DomainGroupMember` | Додати себе до групи |
| `Add Members` | `Add-DomainGroupMember` | Додавати членів до групи |

### Сценарії атак через ACL
| Атака | Опис |
|-------|------|
| `Abusing forgot password permissions` | Скидання паролів привілейованих акаунтів |
| `Abusing group membership management` | Додавання себе до привілейованих груп |
| `Excessive user rights` | Використання надмірних прав виданих акаунту |

***

## 🧪 Практика

### Q1 — What type of ACL defines which security principals are granted or denied access to an object?

> DACL (Discretionary Access Control List) — визначає хто має або не має
> доступ до об'єкта. Складається з ACE записів типу Allow/Deny.

✅ **Відповідь: `DACL`**

***

### Q2 — Which ACE entry can be leveraged to perform a targeted Kerberoasting attack?

> GenericAll надає повний контроль над об'єктом. Якщо маємо цей доступ
> над юзером — можемо виконати targeted Kerberoasting атаку.

✅ **Відповідь: `GenericAll`**
```
