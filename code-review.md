# Code Review - Sistema de Aluguel de Carros

## Comentários de Melhoria - Segurança, Padrões e Boas Práticas

---

### 1️⃣ **ClienteService.java - Duplicação de código no mapeamento DTO**

**Arquivo:** `aluguel-carros/src/main/java/br/puc/aluguelcarros/service/ClienteService.java` (linhas 54-74 e 78-100)

🔍 **Sugestão de melhoria:** A lógica de conversão de Cliente para ClienteDTO está duplicada nos métodos `listarTodosDTO()` e `buscarPorIdDTO()`. Seguindo o princípio DRY (Don't Repeat Yourself) e melhorando a manutenibilidade.

**Benefícios da mudança:**
- Redução de código duplicado
- Facilidade para atualizar a lógica de conversão em um único lugar
- Maior facilidade para testes unitários

**Sugestão de implementação:**
Extrair um método privado `converterClienteParaDTO(Cliente c)` que seja reutilizado em ambos os métodos:

```java
@Transactional
private ClienteDTO converterClienteParaDTO(Cliente c) {
    ClienteDTO dto = new ClienteDTO();
    dto.setId(c.getId());
    dto.setNome(c.getNome());
    dto.setEmail(c.getEmail());
    dto.setRole(c.getRole());
    dto.setCpf(c.getCpf());
    dto.setEndereco(c.getEndereco());
    dto.setProfissao(c.getProfissao());
    dto.setRendimentos(
        c.getRendimentos().stream().map(this::converterRendimentoParaDTO)
            .collect(Collectors.toList())
    );
    return dto;
}

private ClienteDTO.RendimentoDTO converterRendimentoParaDTO(Rendimento r) {
    ClienteDTO.RendimentoDTO rd = new ClienteDTO.RendimentoDTO();
    rd.setId(r.getId());
    rd.setTipo(r.getTipo());
    rd.setValor(r.getValor());
    rd.setComprovante(r.getComprovante());
    return rd;
}
```

---

### 2️⃣ **Tratamento de exceções genérico**

**Arquivo:** `ClienteService.java` (linhas 80, 104), `AuthService.java` (linha 51), `PedidoService.java` (linhas 53, 91, 97)

🔍 **Sugestão de melhoria:** O código lança `RuntimeException` de forma genérica em vários lugares. Isso dificulta o tratamento específico de diferentes tipos de erro no controller.

**Benefícios da mudança:**
- Controllers podem retornar HTTP status codes apropriados (400, 404, 409)
- Melhor logging e rastreamento de erros específicos
- API mais intuitiva para clientes

**Sugestão de implementação:**
Criar exceções personalizadas:

```java
public class RecursoNaoEncontradoException extends RuntimeException {
    public RecursoNaoEncontradoException(String mensagem) {
        super(mensagem);
    }
}

public class RecursoJaExisteException extends RuntimeException {
    public RecursoJaExisteException(String mensagem) {
        super(mensagem);
    }
}
```

Usá-las nos services:
```java
Cliente cliente = repository.findById(id)
    .orElseThrow(() -> new RecursoNaoEncontradoException("Cliente não encontrado: " + id));
```

Criar um `@ControllerAdvice` para mapear essas exceções a HTTP responses:
```java
@ControllerAdvice
public class ExceptionHandler {
    
    @ExceptionHandler(RecursoNaoEncontradoException.class)
    public HttpResponse<Map<String, String>> handleNotFound(RecursoNaoEncontradoException e) {
        return HttpResponse.notFound();
    }
    
    @ExceptionHandler(RecursoJaExisteException.class)
    public HttpResponse<Map<String, String>> handleConflict(RecursoJaExisteException e) {
        return HttpResponse.status(HttpStatus.CONFLICT);
    }
}
```

---

### 3️⃣ **Validação de entrada insuficiente**

**Arquivo:** `ClienteController.java` (linhas 72-74), `PedidoController.java` (linhas 49-51)

🔍 **Sugestão de melhoria:** Os endpoints `POST /clientes` e `POST /pedidos` recebem objetos `@Body` sem validação explícita. Não há verificação se campos obrigatórios estão preenchidos.

**Benefícios da mudança:**
- Prevenção de dados inválidos no banco de dados
- Melhor experiência do cliente (feedback claro sobre o que está errado)
- Segurança contra injeção de dados malformados

**Sugestão de implementação:**
Usar anotações JSR-303 no modelo e adicionar `@Valid` no controller:

```java
@Entity
@Table(name = "clientes")
public class Cliente extends Usuario {
    
    @NotNull(message = "Nome é obrigatório")
    @NotBlank(message = "Nome não pode estar vazio")
    @Size(min = 3, max = 150, message = "Nome deve ter entre 3 e 150 caracteres")
    private String nome;
    
    @NotNull(message = "Email é obrigatório")
    @Email(message = "Email deve ser válido")
    private String email;
    
    // ... outros campos
}

// No controller:
@Post
@Secured(SecurityRule.IS_ANONYMOUS)
public HttpResponse<Cliente> cadastrarCliente(@Body @Valid Cliente cliente) {
    return HttpResponse.created(clienteService.cadastrarCliente(cliente));
}
```

---

### 4️⃣ **Enums para status em vez de Strings**

**Arquivo:** `PedidoService.java` (linhas 70, 92, 102, 118), `PedidoAluguel.java`

🔍 **Sugestão de melhoria:** Os status dos pedidos são representados como Strings ("PENDENTE", "CANCELADO", "APROVADO"). Isso é propenso a erros de digitação e dificulta a manutenção.

**Benefícios da mudança:**
- Type safety - o compilador valida valores válidos
- Redução de bugs causados por typos
- Facilidade para adicionar novos estados no futuro

**Sugestão de implementação:**
Criar um enum:

```java
public enum StatusPedido {
    PENDENTE("Aguardando análise"),
    APROVADO("Aprovado e contrato gerado"),
    CANCELADO("Cancelado pelo cliente ou sistema"),
    REJEITADO("Rejeitado na análise financeira");
    
    private final String descricao;
    
    StatusPedido(String descricao) {
        this.descricao = descricao;
    }
    
    public String getDescricao() {
        return descricao;
    }
}
```

Usar no modelo:
```java
@Enumerated(EnumType.STRING)
@Column(columnDefinition = "VARCHAR(20)")
private StatusPedido status;
```

---

### 5️⃣ **Criptografia de senha no endpoint de atualização**

**Arquivo:** `ClienteService.java` (linhas 102-110)

🔍 **Sugestão de melhoria:** O método `atualizarDados()` não criptografa a senha se ela for atualizada. Se o cliente enviar uma nova senha no campo `senhaHash`, ela será armazenada em texto plano.

**Benefícios da mudança:**
- Segurança de dados dos usuários
- Proteção contra vazamento de senhas
- Conformidade com boas práticas de segurança

**Sugestão de implementação:**
Adicionar verificação e criptografia:

```java
public Cliente atualizarDados(Long id, ClienteDTO dados) {
    Cliente cliente = repository.findById(id)
            .orElseThrow(() -> new RecursoNaoEncontradoException("Cliente não encontrado: " + id));
    
    cliente.setNome(dados.getNome());
    cliente.setEmail(dados.getEmail());
    cliente.setEndereco(dados.getEndereco());
    cliente.setProfissao(dados.getProfissao());
    
    // Se a senha foi fornecida, criptografá-la
    if (dados.getSenha() != null && !dados.getSenha().isEmpty()) {
        String hash = BCrypt.hashpw(dados.getSenha(), BCrypt.gensalt());
        cliente.setSenhaHash(hash);
    }
    
    return repository.update(cliente);
}
```

Nota: Usar um DTO específico para updates que não exponha `senhaHash`.

---

### 6️⃣ **Falta de validação de autorização nos endpoints**

**Arquivo:** `ClienteController.java` (linhas 77-85), `PedidoController.java` (linhas 59-63)

🔍 **Sugestão de melhoria:** Os endpoints `PUT /clientes/{id}` e `DELETE /clientes/{id}` não validam se o usuário autenticado é o dono do recurso. Um cliente poderia alterar ou deletar dados de outro cliente.

**Benefícios da mudança:**
- Segurança contra acesso não autorizado
- Proteção da privacidade dos usuários
- Conformidade com o princípio de "least privilege"

**Sugestão de implementação:**
Extrair o ID do usuário autenticado e validar:

```java
@Put("/{id}")
public HttpResponse<Cliente> atualizarCliente(Long id, @Body Cliente dados, Authentication auth) {
    Long usuarioId = (Long) auth.getAttributes().get("id");
    
    // Impedir que um usuário altere dados de outro usuário
    if (!id.equals(usuarioId)) {
        return HttpResponse.forbidden();
    }
    
    return HttpResponse.ok(clienteService.atualizarDados(id, dados));
}
```

---

### 7️⃣ **Falta de paginação em endpoints de listagem**

**Arquivo:** `ClienteController.java` (linhas 44-46), `PedidoController.java` (linhas 45-47)

🔍 **Sugestão de melhoria:** Os endpoints GET `/clientes` e `GET /pedidos/todos` retornam ALL registros. Com crescimento do banco de dados, isso causará problemas de performance.

**Benefícios da mudança:**
- Melhor performance em grandes datasets
- Redução no uso de memória e bandwidth
- Experiência do usuário mais responsiva

**Sugestão de implementação:**
Adicionar parâmetros de paginação:

```java
@Get
public Page<ClienteDTO> listarClientes(
    @QueryValue(defaultValue = "0") int page,
    @QueryValue(defaultValue = "10") int size
) {
    // Implementar paginação no repository/service
    return clienteService.listarTodosDTO(PageRequest.of(page, size));
}
```

---

### 8️⃣ **Falta de validação de disponibilidade do automóvel**

**Arquivo:** `PedidoService.java` (linhas 50-56)

🔍 **Sugestão de melhoria:** O método `criarSolicitacao()` valida se o automóvel existe e está disponível, mas não há garantia de que entre a validação e o save, outro pedido não tenha reservado o carro (race condition).

**Benefícios da mudança:**
- Evita double-booking de automóveis
- Garantia de integridade dos dados
- Melhor experiência do usuário (sem conflitos de reserva)

**Sugestão de implementação:**
Usar lock pessimista no banco de dados ou implementar lógica de transação:

```java
@Transactional
public PedidoDTO criarSolicitacao(PedidoAluguel pedido) {
    // SELECT ... FOR UPDATE para garantir exclusividade
    Automovel auto = autoRepo.findByIdForUpdate(pedido.getAutomovel().getId())
            .orElseThrow(() -> new RecursoNaoEncontradoException("Automóvel não encontrado"));
    
    if (!auto.isDisponivel()) {
        throw new RecursoIndisponivelException("Automóvel não disponível para aluguel");
    }
    
    // ... resto da lógica
}
```

---

### 9️⃣ **Código hardcoded em método de negócio**

**Arquivo:** `PedidoService.java` (linhas 67-68, 124), `FinanceiroService.java` (linha 56)

🔍 **Sugestão de melhoria:** Valores mágicos como "fator de segurança = 3" e textos hardcoded estão espalhados no código.

**Benefícios da mudança:**
- Facilita ajustes de regras de negócio sem recompilação
- Configurabilidade sem código mágico
- Melhor documentação das regras

**Sugestão de implementação:**
Extrair para constantes ou arquivo de propriedades:

```java
// Application.properties ou application.yml
aluguel-carros.financeiro.fator-seguranca=3.0
aluguel-carros.pedido.dias-minimo-calculo=1
aluguel-carros.contrato.termos-padrao=Contrato de aluguel conforme regulamento...

@Singleton
public class FinanceiroService {
    
    @Value("${aluguel-carros.financeiro.fator-seguranca}")
    private Double fatorSeguranca;
    
    @Transactional
    public boolean realizarAnaliseFinanceira(Long pedidoId) {
        // ... usa this.fatorSeguranca
        return rendimentoTotal >= (pedido.getValorTotal() * fatorSeguranca);
    }
}
```

---

### 🔟 **Método /me retorna Map genérico em vez de DTO**

**Arquivo:** `ClienteController.java` (linhas 49-59)

🔍 **Sugestão de melhoria:** O endpoint GET `/clientes/me` retorna um `Map<String, Object>` genérico. Isso não oferece type safety e é inconsistente com outros endpoints que retornam DTOs.

**Benefícios da mudança:**
- Type safety no cliente (frontend sabe exatamente quais campos existem)
- Documentação automática via Swagger/OpenAPI
- Facilita validação e testes

**Sugestão de implementação:**
Criar um DTO específico:

```java
@Serdeable
public class UsuarioPerfilDTO {
    private Long id;
    private String email;
    private String nome;
    private String role;
    
    // getters e setters
}

// No controller:
@Get("/me")
public HttpResponse<UsuarioPerfilDTO> meuPerfil(Authentication auth) {
    if (auth == null) {
        return HttpResponse.unauthorized();
    }
    
    UsuarioPerfilDTO perfil = new UsuarioPerfilDTO();
    perfil.setId((Long) auth.getAttributes().get("id"));
    perfil.setEmail(auth.getName());
    perfil.setNome((String) auth.getAttributes().get("nome"));
    perfil.setRole((String) auth.getAttributes().get("role"));
    
    return HttpResponse.ok(perfil);
}
```

---

### 1️⃣1️⃣ **Falta de logging em operações críticas**

**Arquivo:** Todos os services (`ClienteService`, `PedidoService`, `FinanceiroService`, `AuthService`)

🔍 **Sugestão de melhoria:** Não há logs das operações realizadas. Dificulta auditoria, debugging e monitoramento do sistema.

**Benefícios da mudança:**
- Rastreamento de todas as operações de negócio
- Facilita debugging em produção
- Conformidade com requisitos de auditoria
- Alertas em caso de anomalias

**Sugestão de implementação:**
Adicionar SLF4J:

```java
@Singleton
public class ClienteService {
    
    private static final Logger logger = LoggerFactory.getLogger(ClienteService.class);
    private final ClienteRepository repository;
    
    public Cliente cadastrarCliente(Cliente cliente) {
        logger.info("Cadastrando novo cliente: {}", cliente.getEmail());
        try {
            String hash = BCrypt.hashpw(cliente.getSenhaHash(), BCrypt.gensalt());
            cliente.setSenhaHash(hash);
            Cliente salvo = repository.save(cliente);
            logger.info("Cliente cadastrado com sucesso. ID: {}", salvo.getId());
            return salvo;
        } catch (Exception e) {
            logger.error("Erro ao cadastrar cliente: {}", cliente.getEmail(), e);
            throw e;
        }
    }
}
```

---

### 1️⃣2️⃣ **Falta de índices no banco de dados**

**Arquivo:** `Cliente.java`, `Usuario.java`, outros modelos

🔍 **Sugestão de melhoria:** O campo `email` é único mas sem índice explícito. Não há índices em campos que são frequentemente consultados (como `clienteId` em `pedidos`).

**Benefícios da mudança:**
- Queries mais rápidas
- Melhor performance em buscar por email
- Melhor performance em relacionamentos

**Sugestão de implementação:**
Adicionar @Index e @UniqueConstraint:

```java
@Entity
@Table(
    name = "clientes",
    uniqueConstraints = @UniqueConstraint(columnNames = "email"),
    indexes = @Index(columnList = "email")
)
public class Cliente extends Usuario {
    
    @Column(nullable = false, unique = true)
    @Index
    private String email;
}

@Entity
@Table(
    name = "pedidos_aluguel",
    indexes = {
        @Index(columnList = "cliente_id"),
        @Index(columnList = "automovel_id"),
        @Index(columnList = "status")
    }
)
public class PedidoAluguel {
    // ...
}
```

---

### 1️⃣3️⃣ **Falta de testes unitários**

**Arquivo:** Todo o projeto

🔍 **Sugestão de melhoria:** Não há testes visíveis no repositório. Isso dificulta manutenção futura e regressões.

**Benefícios da mudança:**
- Confiança ao refatorar código
- Documentação viva do comportamento esperado
- Detecção precoce de bugs
- Facilita integração contínua

**Sugestão de implementação:**
Criar testes para casos críticos:

```java
// src/test/java/.../service/ClienteServiceTest.java

@MicronautTest
public class ClienteServiceTest {
    
    @Inject
    private ClienteService clienteService;
    
    @Inject
    private ClienteRepository clienteRepository;
    
    @Test
    public void deveCriptografarSenhaAoCadastrar() {
        Cliente cliente = new Cliente();
        cliente.setEmail("teste@example.com");
        cliente.setNome("Teste");
        cliente.setSenhaHash("senha123");
        
        Cliente salvo = clienteService.cadastrarCliente(cliente);
        
        assertNotEquals("senha123", salvo.getSenhaHash());
        assertTrue(BCrypt.checkpw("senha123", salvo.getSenhaHash()));
    }
    
    @Test
    public void deveRetornarOptionalVazioParaClienteNaoEncontrado() {
        Optional<Cliente> resultado = clienteService.buscarPorId(999L);
        assertTrue(resultado.isEmpty());
    }
}
```

---

### 1️⃣4️⃣ **Falta de documentação OpenAPI/Swagger**

**Arquivo:** Controllers

🔍 **Sugestão de melhoria:** A API não possui documentação Swagger/OpenAPI. Dificulta a integração com o frontend e consumo por terceiros.

**Benefícios da mudança:**
- Documentação automática e atualizada
- Interface interativa para testar endpoints
- Facilita integração com frontend
- Padrão de indústria

**Sugestão de implementação:**
Adicionar anotações OpenAPI:

```java
@Controller("/clientes")
@Secured(SecurityRule.IS_AUTHENTICATED)
@Tag(name = "Clientes", description = "Gerenciamento de clientes do sistema")
public class ClienteController {
    
    @Get
    @Operation(
        summary = "Listar todos os clientes",
        description = "Retorna uma lista de todos os clientes cadastrados"
    )
    public List<ClienteDTO> listarClientes() {
        return clienteService.listarTodosDTO();
    }
    
    @Get("/{id}")
    @Operation(summary = "Buscar cliente por ID")
    @ApiResponse(
        responseCode = "200",
        description = "Cliente encontrado"
    )
    @ApiResponse(
        responseCode = "404",
        description = "Cliente não encontrado"
    )
    public HttpResponse<ClienteDTO> buscarCliente(
        @Parameter(description = "ID do cliente") Long id
    ) {
        try {
            return HttpResponse.ok(clienteService.buscarPorIdDTO(id));
        } catch (RuntimeException e) {
            return HttpResponse.notFound();
        }
    }
}
```

---

### 1️⃣5️⃣ **Design pattern: Service Locator desnecessário**

**Arquivo:** Controllers (injeção de múltiplos services)

🔍 **Sugestão de melhoria:** Alguns controllers injetam múltiplos services (`PedidoService`, `FinanceiroService`). Considerar uma façade para simplificar.

**Benefícios da mudança:**
- Redução de acoplamento
- Controllers mais simples
- Facilita testes

**Sugestão de implementação:**
Criar uma façade:

```java
@Singleton
public class PedidoFacade {
    
    private final PedidoService pedidoService;
    private final FinanceiroService financeiroService;
    
    public PedidoFacade(PedidoService pedidoService, FinanceiroService financeiroService) {
        this.pedidoService = pedidoService;
        this.financeiroService = financeiroService;
    }
    
    @Transactional
    public ContratoDTO criarEAnalisarPedido(PedidoAluguel pedido) {
        PedidoDTO pedidoDTO = pedidoService.criarSolicitacao(pedido);
        
        boolean aprovado = financeiroService.realizarAnaliseFinanceira(pedidoDTO.getId());
        if (!aprovado) {
            throw new AnaliseFinanceiraFalhouException("Cliente não atende requisitos financeiros");
        }
        
        return pedidoService.converterParaContrato(pedidoDTO.getId());
    }
}

// No controller:
@Singleton
public class PedidoController {
    
    private final PedidoFacade pedidoFacade;
    
    // ... usa apenas a façade
}
```

---

### 1️⃣6️⃣ **Falta de validação de datas**

**Arquivo:** `PedidoService.java` (linhas 59-64)

🔍 **Sugestão de melhoria:** A validação de datas não verifica se as datas estão no passado ou muito distantes no futuro.

**Benefícios da mudança:**
- Prevenção de pedidos com datas inválidas
- Melhor validação de entrada
- Regras de negócio mais claras

**Sugestão de implementação:**

```java
@Transactional
public PedidoDTO criarSolicitacao(PedidoAluguel pedido) {
    // ... validações existentes ...
    
    // Validar datas
    LocalDate hoje = LocalDate.now();
    if (pedido.getDataInicio().isBefore(hoje)) {
        throw new DatasInvalidasException("Data de início não pode ser no passado");
    }
    
    if (ChronoUnit.DAYS.between(hoje, pedido.getDataInicio()) > 365) {
        throw new DatasInvalidasException("Pedido não pode ser feito com mais de 1 ano de antecedência");
    }
    
    // ... resto da lógica
}
```

---

### 1️⃣7️⃣ **Ausência de soft delete**

**Arquivo:** `ClienteController.java` (linhas 82-85)

🔍 **Sugestão de melhoria:** O método `DELETE` faz delete hard do cliente. Isso pode quebrar referências em pedidos e contratos históricos.

**Benefícios da mudança:**
- Mantém integridade referencial histórica
- Preserva dados para auditoria
- Permite "restore" de dados deletados

**Sugestão de implementação:**
Implementar soft delete:

```java
@Entity
@Table(name = "clientes")
public class Cliente extends Usuario {
    
    @Column(name = "ativo", nullable = false)
    private Boolean ativo = true;
    
    @Column(name = "deletado_em")
    private LocalDateTime deletadoEm;
    
    public void deletarLogicamente() {
        this.ativo = false;
        this.deletadoEm = LocalDateTime.now();
    }
    
    public void restaurar() {
        this.ativo = true;
        this.deletadoEm = null;
    }
}

// No service:
public void excluirCadastro(Long id) {
    Cliente cliente = repository.findById(id)
            .orElseThrow(() -> new RecursoNaoEncontradoException("Cliente não encontrado"));
    cliente.deletarLogicamente();
    repository.update(cliente);
}

// Nos queries, adicionar Where:
@Repository
public interface ClienteRepository extends CrudRepository<Cliente, Long> {
    @Query("SELECT c FROM Cliente c WHERE c.ativo = true AND c.id = :id")
    Optional<Cliente> findById(Long id);
}
```

---

### 1️⃣8️⃣ **Falta de transação em método de conversão para contrato**

**Arquivo:** `PedidoService.java` (linhas 115-135)

🔍 **Sugestão de melhoria:** O método `converterParaContrato()` atualiza o status do pedido E cria um novo contrato em passos separados. Se a criação do contrato falhar, o pedido fica em estado inconsistente.

**Benefícios da mudança:**
- Atomicidade - ou ambas operações sucedem ou ambas falham
- Evita estado inconsistente
- Melhor tratamento de erros

**Sugestão de implementação:**
Já tem `@Transactional`, mas verificar rollback automático:

```java
@Transactional(rollbackOn = Exception.class)
public ContratoDTO converterParaContrato(Long id) {
    PedidoAluguel pedido = repository.findById(id)
            .orElseThrow(() -> new RecursoNaoEncontradoException("Pedido não encontrado: " + id));
    
    if (!"PENDENTE".equals(pedido.getStatus())) {
        throw new EstadoInvalidoException("Apenas pedidos PENDENTE podem ser convertidos em contrato");
    }
    
    try {
        pedido.setStatus("APROVADO");
        repository.update(pedido);
        
        Contrato contrato = new Contrato();
        contrato.setPedidoAluguel(pedido);
        contrato.setValorFinal(pedido.getValorTotal());
        contrato.setTermos("Contrato de aluguel gerado automaticamente.");
        Contrato salvo = contratoRepo.save(contrato);
        
        // ... resto
    } catch (Exception e) {
        logger.error("Erro ao converter pedido para contrato", e);
        throw new ErroProcessamentoException("Falha ao converter pedido para contrato", e);
    }
}
```

---

### 1️⃣9️⃣ **Falta de arquitetura em camadas no frontend**

**Arquivo:** Frontend React

🔍 **Sugestão de melhoria:** Estrutura recomendada para melhor organização:
- **services/api.js** - Chamadas HTTP centralizadas
- **hooks/useClientes.js** - Lógica de estado do cliente
- **components/** - Componentes reutilizáveis
- **pages/** - Páginas completas
- **utils/** - Funções utilitárias

**Benefícios da mudança:**
- Código mais organizado e manutenível
- Fácil reutilização de componentes
- Separação de responsabilidades
- Testes mais simples

---

### 2️⃣0️⃣ **Falta de validação CSRF**

**Arquivo:** `AuthController.java`, `ClienteController.java`

🔍 **Sugestão de melhoria:** Endpoints que modificam dados (POST, PUT, DELETE) devem ter proteção CSRF.

**Benefícios da mudança:**
- Proteção contra ataques CSRF
- Segurança em operações que modificam estado
- Conformidade com boas práticas

**Sugestão de implementação:**
Configurar no Micronaut:

```java
// application.yml
micronaut:
  security:
    csrf:
      enabled: true
      header-name: "X-CSRF-TOKEN"
      cookie-name: "XSRF-TOKEN"
      token-regeneration: true
```

---

### 2️⃣1️⃣ **Falta de rate limiting**

**Arquivo:** Controllers

🔍 **Sugestão de melhoria:** Endpoints de login e registro não têm proteção contra força bruta.

**Benefícios da mudança:**
- Proteção contra ataques de força bruta
- Proteção da API contra abuso
- Melhor segurança

**Sugestão de implementação:**
Usar Micronaut RateLimit ou Guava RateLimiter:

```java
@Singleton
public class RateLimitInterceptor implements HttpServerFilter {
    
    private final LoadingCache<String, RateLimiter> limiters = CacheBuilder.newBuilder()
            .expireAfterAccess(1, TimeUnit.MINUTES)
            .build(new CacheLoader<String, RateLimiter>() {
                public RateLimiter load(String key) {
                    return RateLimiter.create(5.0); // 5 requisições por segundo
                }
            });
    
    @Override
    public Publisher<MutableHttpResponse<?>> doFilter(HttpRequest<?> request, ServerFilterChain chain) {
        String clientIp = request.getRemoteAddress().getHostAddress();
        
        if (!limiters.getUnchecked(clientIp).tryAcquire()) {
            return Flowable.just(HttpResponse.status(HttpStatus.TOO_MANY_REQUESTS));
        }
        
        return chain.proceed(request);
    }
}
```

---

## ✅ Resumo das Melhorias

| # | Categoria | Severidade | Impacto |
|---|-----------|-----------|--------|
| 1 | Code Smell | Média | Manutenibilidade |
| 2 | Exceções | Alta | Tratamento de erros |
| 3 | Validação | Alta | Segurança |
| 4 | Type Safety | Média | Bugs |
| 5 | Segurança | Alta | Dados de usuário |
| 6 | Autorização | Crítica | Segurança |
| 7 | Performance | Média | Escalabilidade |
| 8 | Concorrência | Alta | Data integrity |
| 9 | Config | Média | Manutenibilidade |
| 10 | API Design | Média | Usabilidade |
| 11 | Logging | Média | Observabilidade |
| 12 | Database | Média | Performance |
| 13 | Testing | Alta | Confiabilidade |
| 14 | Documentation | Média | Integração |
| 15 | Architecture | Média | Manutenibilidade |
| 16 | Validação | Média | Qualidade |
| 17 | Data | Alta | Integridade |
| 18 | Transações | Alta | Consistência |
| 19 | Frontend | Média | Manutenibilidade |
| 20 | Security | Crítica | Segurança |
| 21 | Security | Alta | Segurança |

---

## 🎯 Próximos Passos Recomendados

1. **Curto prazo** (Sprint 1):
   - Criar exceções personalizadas
   - Adicionar validação com JSR-303
   - Implementar ControllerAdvice

2. **Médio prazo** (Sprint 2-3):
   - Refatorar duplicação de DTO mappings
   - Adicionar testes unitários
   - Implementar logging

3. **Longo prazo** (Sprint 4+):
   - Documentação OpenAPI
   - Soft delete
   - Rate limiting e CSRF

---

**Data da revisão:** 2026-05-25
**Revisor:** João Victor Z.

