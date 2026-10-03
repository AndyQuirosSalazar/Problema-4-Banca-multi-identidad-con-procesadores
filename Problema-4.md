# Problema 4: Banca multi-identidad con procesadores
 
## El problema
 
En el diseño original todo estaba metido en BankingService, ya que era la clase que revisaba el tipo de identidad, el tipo de operación y a qué procesador llamar, lo cual lo llenaba de if y switch que crecían cada vez que se agregaba algo nuevo, por ejemplo una identidad nueva o un banco nuevo, y eso es lo que provoca el código espagueti.
 
## La solución
 
Lo que hice fue pasar cada decisión a la clase que mejor la conoce. Identity ahora es una clase abstracta y cada tipo (Personal, Business, Minor y ForeignResident) sabe cuál es su límite diario, qué operaciones puede hacer y qué reglas extra tiene. Lo mismo pasa con BankOperation: cada operación sabe cuánto suma en total, por ejemplo Payroll multiplica el monto por la cantidad de empleados y Deposit sabe que no cuenta contra el límite. Con los procesadores creé una interfaz común llamada PaymentProcessor, de modo que BankingService solo coordina los pasos sin saber qué banco está usando ni qué tipo de identidad tiene enfrente.
 
## Cómo se usa el polimorfismo
 
Estas son las únicas líneas donde BankingService aplica el comportamiento polimórfico, ya que la misma llamada se comporta distinto según el objeto que la recibe:
 
```java
String error = identity.validate(operation, today);          // cada identidad aplica sus propias reglas
double limit = identity.dailyLimit();                        // Personal 2000, Business 50000, Minor 100...
double total = operation.totalAmount();                      // Payroll = monto x empleados
OperationResult result = processor.execute(identity, operation); // cada banco responde a su manera
```
 
Por ejemplo, en ForeignResidentIdentity solo se sobreescribe la regla que le corresponde:
 
```java
@Override
public String validate(BankOperation op, LocalDate today) {
    return residencyExpiresOn.isBefore(today) ? "RESIDENCY_EXPIRED" : null;
}
```
 
## Los wrappers de los procesadores
 
Como cada banco es una entidad externa y no se puede cambiar su código, lo que hice fue envolver cada uno en un adapter que implementa PaymentProcessor. Así BankingService siempre llama a execute(...), y es el adapter quien traduce a la forma particular de cada banco y convierte su manera de fallar en un OperationResult. Las clases originales quedan intactas.
 
```java
// NationalBankAdapter: postTransaction devuelve null si rechaza
String ref = bank.postTransaction(from, to, amount, currency);
return ref == null ? OperationResult.rejected("REJECTED_BY_PROCESSOR") : OperationResult.success(ref);
 
// PacificBankAdapter: usa centavos y devuelve "REJECTED:<reason>"
String r = bank.submit(from, to, Math.round(amount * 100), concept);
return r.startsWith("REJECTED:") ? OperationResult.rejected(r.substring(9)) : OperationResult.success(r);
 
// SwiftGatewayAdapter: lanza una excepción si los datos son inválidos
try {
    return OperationResult.success(gateway.sendWire(from, to, bic, amount, currency));
} catch (Exception e) {
    return OperationResult.failed(e.getMessage());
}
```
 
## Diagrama
 
```mermaid
classDiagram
    direction TB
 
    class OperationType {
        <<enumeration>>
        DEPOSIT
        WITHDRAWAL
        DOMESTIC_TRANSFER
        INTERNATIONAL_TRANSFER
        PAYROLL
    }
    class ProcessorType {
        <<enumeration>>
        NATIONAL
        PACIFIC
        SWIFT
    }
 
    class User {
        -String id
        -String fullName
        -List~Identity~ identities
        +addIdentity(Identity)
        +findIdentity(String kind) Identity
    }
 
    class Identity {
        <<abstract>>
        #String documentNumber
        #String accountNumber
        +allowedOperations() Set~OperationType~
        +dailyLimit() double
        +allowsProcessor(ProcessorType) boolean
        +validate(BankOperation, LocalDate) String
    }
    class PersonalIdentity
    class BusinessIdentity {
        -String companyName
    }
    class MinorIdentity {
        -String guardianUserId
    }
    class ForeignResidentIdentity {
        -String countryCode
        -LocalDate residencyExpiresOn
    }
 
    class BankOperation {
        <<abstract>>
        #double amount
        #String approvalCode
        +type() OperationType
        +totalAmount() double
        +countsAgainstDailyLimit() boolean
    }
    class DepositOperation
    class WithdrawalOperation
    class DomesticTransferOperation {
        -String destinationAccount
    }
    class InternationalTransferOperation {
        -String destinationAccount
        -String destinationBic
        -String currency
    }
    class PayrollOperation {
        -List~String~ employeeAccounts
    }
 
    class PaymentProcessor {
        <<interface>>
        +type() ProcessorType
        +supports(OperationType) boolean
        +calculateFee(BankOperation) double
        +execute(Identity, BankOperation) OperationResult
    }
    class NationalBankAdapter {
        -NationalBankProcessor bank
    }
    class PacificBankAdapter {
        -PacificBankProcessor bank
    }
    class SwiftGatewayAdapter {
        -SwiftGatewayProcessor gateway
    }
 
    class NationalBankProcessor {
        <<externo, no se modifica>>
        +postTransaction(String, String, double, String) String
    }
    class PacificBankProcessor {
        <<externo, no se modifica>>
        +submit(String, String, long, String) String
        +submitPayroll(String, List~String~, long) String
    }
    class SwiftGatewayProcessor {
        <<externo, no se modifica>>
        +sendWire(String, String, String, double, String) String
    }
 
    class BankingService {
        -List~PaymentProcessor~ processors
        -DailyUsageTracker usageTracker
        -AuditLog auditLog
        +execute(User, String kind, BankOperation, LocalDate) OperationResult
        -selectProcessor(Identity, BankOperation) PaymentProcessor
    }
    class DailyUsageTracker {
        +usedOn(String account, LocalDate) double
        +record(String account, LocalDate, double)
    }
    class AuditLog {
        +record(String entry)
    }
    class OperationResult {
        -boolean success
        -String reference
        -String reason
        -double fee
        +success(String) OperationResult
        +rejected(String) OperationResult
        +failed(String) OperationResult
    }
 
    Identity <|-- PersonalIdentity
    Identity <|-- BusinessIdentity
    Identity <|-- MinorIdentity
    Identity <|-- ForeignResidentIdentity
 
    BankOperation <|-- DepositOperation
    BankOperation <|-- WithdrawalOperation
    BankOperation <|-- DomesticTransferOperation
    BankOperation <|-- InternationalTransferOperation
    BankOperation <|-- PayrollOperation
 
    PaymentProcessor <|.. NationalBankAdapter
    PaymentProcessor <|.. PacificBankAdapter
    PaymentProcessor <|.. SwiftGatewayAdapter
 
    NationalBankAdapter --> NationalBankProcessor : envuelve postTransaction
    PacificBankAdapter --> PacificBankProcessor : envuelve submit y submitPayroll
    SwiftGatewayAdapter --> SwiftGatewayProcessor : envuelve sendWire
 
    User "1" *-- "1..*" Identity
    BankOperation --> OperationType
    PaymentProcessor --> ProcessorType
    BankingService o-- "3" PaymentProcessor
    BankingService *-- DailyUsageTracker
    BankingService *-- AuditLog
    BankingService ..> OperationResult : devuelve