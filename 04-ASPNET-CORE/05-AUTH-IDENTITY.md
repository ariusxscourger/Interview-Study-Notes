# AUTHENTICATION & IDENTITY
## Exam-Style Study Notes

---

## TOPIC: ASP.NET Core Authentication & Identity

### WHY? (Problem Statement)

**Before Modern Auth:**
- Custom username/password in DB → Plain text or weak hashing
- Session cookies only → No token-based auth for APIs
- No standards → Every app reinvented security
- No social login → Users hate creating accounts
- No MFA → Single point of failure

**What ASP.NET Core Identity + Auth Solves:**
| Problem | Solution |
|---------|----------|
| Password storage | PBKDF2/BCrypt/Argon2 hashing built-in |
| Token standards | JWT Bearer, OpenID Connect, OAuth 2.0 |
| Social login | Google, Microsoft, Facebook, GitHub providers |
| MFA | TOTP (Authenticator apps), SMS, Email |
| Account management | Email confirmation, password reset, lockout |
| API Security | `[Authorize]`, Policies, Roles, Claims |
| Session management | Sliding expiration, revocation |

**Real-World Analogy:**
- Old Auth = Handwritten ID card (easy to forge, no standards)
- Modern Auth = Biometric passport (cryptographically verified, international standard)

---

### HOW? (Internal Mechanism)

#### 1. **Authentication Flow (JWT Bearer)**

```
┌─────────┐     1. Login      ┌──────────────┐     2. Token      ┌─────────┐
│ Client  │ ─────────────────► │ Auth Server  │ ────────────────► │ Client  │
│         │  (username/pass)  │ (Identity)   │  (JWT + Refresh)  │         │
└─────────┘                    └──────────────┘                   └─────────┘
      │                                                            │
      │ 3. Request + JWT                                          │
      ▼                                                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        RESOURCE SERVER (API)                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  UseAuthentication()                                            │   │
│  │    └─► JwtBearerHandler.ValidateToken()                         │   │
│  │         ├─ Signature validation (RSA/ECDSA/HMAC)                │   │
│  │         ├─ Expiration check (exp claim)                         │   │
│  │         ├─ Issuer/Audience validation                           │   │
│  │         └─ TokenReplayCache (optional)                          │   │
│  │                                                                 │   │
│  │  Sets: HttpContext.User = ClaimsPrincipal                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                    │
│                                    ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  UseAuthorization()                                             │   │
│  │    └─► Policy/Role/Claim evaluation                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2. **Identity System Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│                      ASP.NET Core Identity                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  UserManager<TUser>          SignInManager<TUser>               │
│  ├─ CreateAsync()            ├─ PasswordSignInAsync()           │
│  ├─ FindByEmailAsync()       ├─ SignInWithClaimsAsync()         │
│  ├─ CheckPasswordAsync()     ├─ TwoFactorSignInAsync()          │
│  ├─ AddClaimAsync()          └─ SignOutAsync()                  │
│  ├─ AddToRoleAsync()                                                 │
│  ├─ GenerateEmailConfirmationTokenAsync()                         │
│  ├─ ResetPasswordAsync()                                          │
│  └─ UpdateSecurityStampAsync()                                    │
│                                                                 │
│  RoleManager<TRole>         IUserStore<TUser>                    │
│  ├─ CreateAsync()           ├─ FindByIdAsync()                  │
│  ├─ AddClaimAsync()         ├─ FindByNameAsync()                │
│  └─ RoleExistsAsync()       ├─ CreateAsync()                    │
│                              └─ ... (IUserPasswordStore,        │
│                                  IUserEmailStore,                │
│                                  IUserLockoutStore,              │
│                                  IUserTwoFactorStore)            │
└─────────────────────────────────────────────────────────────────┘
```

#### 3. **JWT Token Structure**

```
HEADER (Base64Url)
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-123"
}
.
PAYLOAD (Base64Url)
{
  "sub": "user-id-123",
  "email": "user@nanolite.com",
  "name": "Saqib",
  "roles": ["Admin", "Developer"],
  "permissions": ["orders.read", "orders.write"],
  "iat": 1700000000,
  "exp": 1700003600,
  "iss": "https://auth.nanolite.com",
  "aud": "https://api.nanolite.com"
}
.
SIGNATURE (Base64Url)
RSA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), privateKey)
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Identity Setup (Program.cs)**

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1. DbContext
builder.Services.AddDbContext<AppDbContext>(opt => 
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// 2. Identity with custom User/Role
builder.Services.AddIdentity<AppUser, AppRole>(options =>
{
    // Password settings
    options.Password.RequireDigit = true;
    options.Password.RequiredLength = 8;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireLowercase = true;
    
    // Lockout
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;
    
    // User
    options.User.RequireUniqueEmail = true;
    options.User.AllowedUserNameCharacters = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-._@+";
    
    // SignIn
    options.SignIn.RequireConfirmedEmail = true;
    options.SignIn.RequireConfirmedPhoneNumber = false;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders()  // Email confirmation, password reset tokens
.AddTokenProvider<DataProtectorTokenProvider<AppUser>>("Custom");

// 3. JWT Authentication
var jwtSettings = builder.Configuration.GetSection("Jwt").Get<JwtSettings>();
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("Jwt"));

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtSettings.Issuer,
        ValidAudience = jwtSettings.Audience,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSettings.SecretKey)),
        ClockSkew = TimeSpan.Zero,  // No grace period
        RoleClaimType = ClaimTypes.Role,
        NameClaimType = ClaimTypes.Name
    };
    
    // Optional: Read token from cookie for Blazor/Web
#/SPA
    options.Events = new JwtBearerEvents
    {
        OnMessageReceived = context =>
        {
            var token = context.Request.Cookies["access_token"];
            if (!string.IsNullOrEmpty(token))
                context.Token = token;
            return Task.CompletedTask;
        }
    };
});

// 4. Authorization Policies
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdmin", p => p.RequireRole("Admin"));
    options.AddPolicy("RequireDeveloper", p => p.RequireRole("Developer", "Admin"));
    options.AddPolicy("OrderRead", p => p.RequireClaim("permission", "orders.read"));
    options.AddPolicy("OrderWrite", p => p.RequireClaim("permission", "orders.write"));
    options.AddPolicy("Over18", p => p.RequireAssertion(ctx => 
    {
        var dobClaim = ctx.User.FindFirst("date_of_birth");
        return dobClaim != null && DateTime.TryParse(dobClaim.Value, out var dob) 
               && (DateTime.UtcNow - dob).TotalDays > 18 * 365;
    }));
});

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
```

#### 2. **Custom User & Role Entities**

```csharp
// Domain Layer
public class AppUser : IdentityUser<Guid>
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime? DateOfBirth { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }
    
    // Navigation
    public ICollection<AppUserRole> UserRoles { get; set; } = new List<AppUserRole>();
    public ICollection<RefreshToken> RefreshTokens { get; set; } = new List<RefreshToken>();
}

public class AppRole : IdentityRole<Guid>
{
    public string Description { get; set; }
    public ICollection<AppUserRole> UserRoles { get; set; } = new List<AppUserRole>();
}

public class AppUserRole : IdentityUserRole<Guid>
{
    public AppUser User { get; set; }
    public AppRole Role { get; set; }
}

// Refresh Token for token rotation
public class RefreshToken
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }
    public string Token { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? RevokedAt { get; set; }
    public string? ReplacedByToken { get; set; }
    public AppUser User { get; set; }
    
    public bool IsActive => RevokedAt == null && DateTime.UtcNow < ExpiresAt;
}

// DbContext Configuration
public class AppDbContext : IdentityDbContext<AppUser, AppRole, Guid, 
    IdentityUserClaim<Guid>, AppUserRole, IdentityUserLogin<Guid>, 
    IdentityRoleClaim<Guid>, IdentityUserToken<Guid>>
{
    public DbSet<RefreshToken> RefreshTokens { get; set; }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        builder.Entity<AppUser>(b =>
        {
            b.ToTable("Users");
            b.Property(u => u.FirstName).HasMaxLength(100);
            b.Property(u => u.LastName).HasMaxLength(100);
        });
        
        builder.Entity<AppRole>(b => b.ToTable("Roles"));
        builder.Entity<AppUserRole>(b => b.ToTable("UserRoles"));
        builder.Entity<IdentityUserClaim<Guid>>(b => b.ToTable("UserClaims"));
        builder.Entity<IdentityUserLogin<Guid>>(b => b.ToTable("UserLogins"));
        builder.Entity<IdentityRoleClaim<Guid>>(b => b.ToTable("RoleClaims"));
        builder.Entity<IdentityUserToken<Guid>>(b => b.ToTable("UserTokens"));
        
        builder.Entity<RefreshToken>(b =>
        {
            b.ToTable("RefreshTokens");
            b.HasKey(rt => rt.Id);
            b.HasIndex(rt => rt.Token).IsUnique();
            b.HasOne(rt => rt.User).WithMany(u => u.RefreshTokens).HasForeignKey(rt => rt.UserId);
        });
    }
}
```

#### 3. **Auth Service Implementation**

```csharp
public interface IAuthService
{
    Task<AuthResult> RegisterAsync(RegisterDto dto);
    Task<AuthResult> LoginAsync(LoginDto dto);
    Task<AuthResult> RefreshTokenAsync(string refreshToken);
    Task<bool> RevokeTokenAsync(string refreshToken);
    Task<bool> ConfirmEmailAsync(Guid userId, string token);
    Task<bool> ForgotPasswordAsync(string email);
    Task<bool> ResetPasswordAsync(ResetPasswordDto dto);
    Task<bool> ChangePasswordAsync(Guid userId, ChangePasswordDto dto);
    Task<bool> EnableMfaAsync(Guid userId);
    Task<bool> VerifyMfaAsync(Guid userId, string code);
}

public class AuthService : IAuthService
{
    private readonly UserManager<AppUser> _userManager;
    private readonly SignInManager<AppUser> _signInManager;
    private readonly IJwtTokenGenerator _jwtGenerator;
    private readonly AppDbContext _db;
    private readonly IEmailService _emailService;
    
    public async Task<AuthResult> RegisterAsync(RegisterDto dto)
    {
        var existingUser = await _userManager.FindByEmailAsync(dto.Email);
        if (existingUser != null)
            return AuthResult.Failure("Email already registered");
        
        var user = new AppUser
        {
            UserName = dto.Email,
            Email = dto.Email,
            FirstName = dto.FirstName,
            LastName = dto.LastName,
            DateOfBirth = dto.DateOfBirth,
            EmailConfirmed = false
        };
        
        var result = await _userManager.CreateAsync(user, dto.Password);
        if (!result.Succeeded)
            return AuthResult.Failure(result.Errors.Select(e => e.Description));
        
        // Assign default role
        await _userManager.AddToRoleAsync(user, "User");
        
        // Send confirmation email
        var token = await _userManager.GenerateEmailConfirmationTokenAsync(user);
        await _emailService.SendConfirmationEmailAsync(user.Email, token);
        
        return AuthResult.Success("Registration successful. Check email to confirm.");
    }
    
    public async Task<AuthResult> LoginAsync(LoginDto dto)
    {
        var user = await _userManager.FindByEmailAsync(dto.Email);
        if (user == null || !user.IsActive)
            return AuthResult.Failure("Invalid credentials");
        
        if (!user.EmailConfirmed)
            return AuthResult.Failure("Email not confirmed");
        
        var result = await _signInManager.CheckPasswordSignInAsync(user, dto.Password, lockoutOnFailure: true);
        if (result.IsLockedOut)
            return AuthResult.Failure("Account locked. Try again later.");
        if (!result.Succeeded)
            return AuthResult.Failure("Invalid credentials");
        
        // Update last login
        user.LastLoginAt = DateTime.UtcNow;
        await _userManager.UpdateAsync(user);
        
        // Generate tokens
        var accessToken = await _jwtGenerator.GenerateAccessTokenAsync(user);
        var refreshToken = _jwtGenerator.GenerateRefreshToken();
        
        // Store refresh token
        user.RefreshTokens.Add(new RefreshToken
        {
            Token = refreshToken,
            ExpiresAt = DateTime.UtcNow.AddDays(7)
        });
        await _db.SaveChangesAsync();
        
        return AuthResult.Success(accessToken, refreshToken, user);
    }
    
    public async Task<AuthResult> RefreshTokenAsync(string refreshToken)
    {
        var storedToken = await _db.RefreshTokens
            .Include(rt => rt.User)
            .FirstOrDefaultAsync(rt => rt.Token == refreshToken);
        
        if (storedToken == null || !storedToken.IsActive)
            return AuthResult.Failure("Invalid or expired refresh token");
        
        // Revoke old token (rotation)
        storedToken.RevokedAt = DateTime.UtcNow;
        
        // Generate new tokens
        var newAccessToken = await _jwtGenerator.GenerateAccessTokenAsync(storedToken.User);
        var newRefreshToken = _jwtGenerator.GenerateRefreshToken();
        
        storedToken.User.RefreshTokens.Add(new RefreshToken
        {
            Token = newRefreshToken,
            ExpiresAt = DateTime.UtcNow.AddDays(7)
        });
        
        storedToken.ReplacedByToken = newRefreshToken;
        await _db.SaveChangesAsync();
        
        return AuthResult.Success(newAccessToken, newRefreshToken, storedToken.User);
    }
}
```

#### 4. **JWT Token Generator**

```csharp
public interface IJwtTokenGenerator
{
    Task<string> GenerateAccessTokenAsync(AppUser user);
    string GenerateRefreshToken();
}

public class JwtTokenGenerator : IJwtTokenGenerator
{
    private readonly JwtSettings _settings;
    private readonly UserManager<AppUser> _userManager;
    
    public JwtTokenGenerator(IOptions<JwtSettings> settings, UserManager<AppUser> userManager)
    {
        _settings = settings.Value;
        _userManager = userManager;
    }
    
    public async Task<string> GenerateAccessTokenAsync(AppUser user)
    {
        var roles = await _userManager.GetRolesAsync(user);
        var claims = await _userManager.GetClaimsAsync(user);
        
        var tokenClaims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new(JwtRegisteredClaimNames.Name, user.UserName!),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(), ClaimValueTypes.Integer64)
        };
        
        // Add roles
        tokenClaims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));
        
        // Add custom claims
        tokenClaims.AddRange(claims);
        
        // Add permissions from roles
        var permissions = await GetPermissionsForRoles(roles);
        tokenClaims.AddRange(permissions.Select(p => new Claim("permission", p)));
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.SecretKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: tokenClaims,
            expires: DateTime.UtcNow.AddMinutes(_settings.ExpiryMinutes),
            signingCredentials: creds
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
    
    private async Task<List<string>> GetPermissionsForRoles(IList<string> roles)
    {
        // Map roles to permissions (could be from DB)
        var rolePermissions = new Dictionary<string, List<string>>
        {
            ["Admin"] = new() { "orders.read", "orders.write", "users.read", "users.write", "admin.access" },
            ["Developer"] = new() { "orders.read", "orders.write", "users.read" },
            ["User"] = new() { "orders.read" }
        };
        
        return roles.SelectMany(r => rolePermissions.GetValueOrDefault(r, new()))
                   .Distinct()
                   .ToList();
    }
}
```

#### 5. **Controller / Minimal API Endpoints**

```csharp
// Minimal API
var authGroup = app.MapGroup("/api/auth").WithTags("Auth");

authGroup.MapPost("/register", async (RegisterDto dto, IAuthService auth) =>
{
    var result = await auth.RegisterAsync(dto);
    return result.Succeeded ? Results.Ok(result) : Results.BadRequest(result);
});

authGroup.MapPost("/login", async (LoginDto dto, IAuthService auth) =>
{
    var result = await auth.LoginAsync(dto);
    return result.Succeeded ? Results.Ok(result) : Results.Unauthorized();
});

authGroup.MapPost("/refresh", async (RefreshTokenDto dto, IAuthService auth) =>
{
    var result = await auth.RefreshTokenAsync(dto.RefreshToken);
    return result.Succeeded ? Results.Ok(result) : Results.Unauthorized();
});

authGroup.MapPost("/revoke", [Authorize] async (RefreshTokenDto dto, IAuthService auth) =>
{
    await auth.RevokeTokenAsync(dto.RefreshToken);
    return Results.Ok();
});

authGroup.MapPost("/forgot-password", async (ForgotPasswordDto dto, IAuthService auth) =>
{
    await auth.ForgotPasswordAsync(dto.Email);
    return Results.Ok("If email exists, reset link sent");
});

authGroup.MapPost("/reset-password", async (ResetPasswordDto dto, IAuthService auth) =>
{
    var result = await auth.ResetPasswordAsync(dto);
    return result ? Results.Ok() : Results.BadRequest("Invalid token");
});

// Protected endpoints
var api = app.MapGroup("/api").RequireAuthorization();

api.MapGet("/orders", [Authorize(Policy = "OrderRead")] async (IOrderService svc) =>
    await svc.GetOrdersAsync());

api.MapPost("/orders", [Authorize(Policy = "OrderWrite")] async (CreateOrderDto dto, IOrderService svc) =>
    await svc.CreateOrderAsync(dto));

// MFA Endpoints
authGroup.MapPost("/mfa/enable", [Authorize] async (HttpContext ctx, IAuthService auth) =>
{
    var userId = Guid.Parse(ctx.User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    var result = await auth.EnableMfaAsync(userId);
    return result ? Results.Ok() : Results.BadRequest();
});

authGroup.MapPost("/mfa/verify", [Authorize] async (MfaVerifyDto dto, HttpContext ctx, IAuthService auth) =>
{
    var userId = Guid.Parse(ctx.User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    var result = await auth.VerifyMfaAsync(userId, dto.Code);
    return result ? Results.Ok() : Results.BadRequest("Invalid code");
});
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Refresh Token Rotation** | Long-lived sessions | Long-lived JWT only | Token theft = permanent access |
| **Short JWT (15min) + Refresh** | Web/SPA | JWT with 24h expiry | No revocation possible |
| **Claims-based Auth** | Fine-grained permissions | Role-only checks | Can't express "owner of resource" |
| **Policy-based Authorization** | Complex rules | `[Authorize(Roles="Admin")]` everywhere | Inflexible, scattered logic |
| **Token in HttpOnly Cookie** | Browser apps | Token in localStorage | XSS steals localStorage |
| **Email Confirmation** | Public registration | Auto-confirm emails | Fake accounts, spam |
| **MFA** | Admin/sensitive actions | Password only | Credential stuffing |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Explain the difference between Authentication and Authorization"

**Answer:**
```
Authentication (AuthN) = WHO ARE YOU?
- Verifies identity
- Methods: Password, MFA, Social, Certificate, Windows Auth
- Result: ClaimsPrincipal with Claims (sub, email, name, roles)

Authorization (AuthZ) = WHAT CAN YOU DO?
- Checks permissions AFTER authentication
- Methods: Roles, Claims, Policies, Requirements
- Result: Allow/Deny/Challenge

Order: AuthN → AuthZ (Always)
Middleware: UseAuthentication() → UseAuthorization()
```

#### 2. **Code**: "Implement JWT token validation manually (without middleware)"

```csharp
public class JwtValidator
{
    private readonly TokenValidationParameters _params;
    
    public JwtValidator(JwtSettings settings)
    {
        _params = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = settings.Issuer,
            ValidAudience = settings.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(settings.SecretKey)),
            ClockSkew = TimeSpan.Zero
        };
    }
    
    public ClaimsPrincipal ValidateToken(string token)
    {
        var handler = new JwtSecurityTokenHandler();
        
        try
        {
            var principal = handler.ValidateToken(token, _params, out var validatedToken);
            
            // Additional checks
            if (validatedToken is not JwtSecurityToken jwt)
                throw new SecurityTokenException("Invalid token type");
            
            if (jwt.Header.Alg != SecurityAlgorithms.HmacSha256)
                throw new SecurityTokenException("Invalid algorithm");
            
            return principal;
        }
        catch (SecurityTokenExpiredException)
        {
            throw new UnauthorizedAccessException("Token expired");
        }
        catch (SecurityTokenException ex)
        {
            throw new UnauthorizedAccessException($"Invalid token: {ex.Message}");
        }
    }
}
```

#### 3. **Design**: "How to implement 'User can only edit their own orders'?"

```csharp
// 1. Resource-based Policy (Requirement + Handler)
public class OrderOwnerRequirement : IAuthorizationRequirement { }

public class OrderOwnerHandler : AuthorizationHandler<OrderOwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        OrderOwnerRequirement requirement, 
        Order resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (resource.CustomerId.ToString() == userId)
            context.Succeed(requirement);
        return Task.CompletedTask;
    }
}

// 2. Register
builder.Services.AddSingleton<IAuthorizationHandler, OrderOwnerHandler>();
builder.Services.AddAuthorization(opt => 
    opt.AddPolicy("OrderOwner", p => p.Requirements.Add(new OrderOwnerRequirement())));

// 3. Use in Controller
[HttpPut("{id}")]
[Authorize(Policy = "OrderOwner")]
public async Task<IActionResult> UpdateOrder(int id, UpdateOrderDto dto)
{
    var order = await _svc.GetOrderAsync(id);
    if (order == null) return NotFound();
    
    // Policy already checked by middleware
    return Ok(await _svc.UpdateOrderAsync(id, dto));
}

// 4. Use in Minimal API
app.MapPut("/api/orders/{id}", [Authorize(Policy = "OrderOwner")] 
    async (int id, UpdateOrderDto dto, IOrderService svc) => 
    await svc.UpdateOrderAsync(id, dto));
```

#### 4. **Security**: "How to prevent JWT token replay attacks?"

**Answer:**
```
1. Short Expiry (15 min) + Refresh Token Rotation
2. Token Replay Cache (Distributed Cache)
   - Store JWT ID (jti) + expiry in Redis
   - Reject if jti already seen
3. One-time Use Tokens (for sensitive ops)
4. mTLS / Certificate Binding (advanced)
5. DPoP (Demonstrating Proof of Possession) - RFC 9449

Implementation:
public class TokenReplayCache
{
    private readonly IDistributedCache _cache;
    
    public async Task<bool> IsReplayAsync(string jti, TimeSpan validFor)
    {
        var key = $"replay:{jti}";
        var exists = await _cache.GetAsync(key);
        if (exists != null) return true;
        
        await _cache.SetAsync(key, Encoding.UTF8.GetBytes("1"), 
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = validFor });
        return false;
    }
}

// In JwtBearerEvents
options.Events = new JwtBearerEvents
{
    OnTokenValidated = async context =>
    {
        var jti = context.Principal.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
        var exp = context.Principal.FindFirst(JwtRegisteredClaimNames.Exp)?.Value;
        
        if (!string.IsNullOrEmpty(jti) && long.TryParse(exp, out var expUnix))
        {
            var cache = context.HttpContext.RequestServices.GetRequiredService<TokenReplayCache>();
            var validFor = DateTimeOffset.FromUnixTimeSeconds(expUnix) - DateTimeOffset.UtcNow;
            
            if (await cache.IsReplayAsync(jti, validFor))
                context.Fail("Token replay detected");
        }
    }
};
```

#### 5. **Code**: "Implement password reset with secure token"

```csharp
public class PasswordResetService
{
    private readonly UserManager<AppUser> _userManager;
    private readonly IEmailService _email;
    
    public async Task<Result> RequestResetAsync(string email)
    {
        var user = await _userManager.FindByEmailAsync(email);
        if (user == null) return Result.Success(); // Don't reveal user existence
        
        // Generate token (valid 1 hour by default)
        var token = await _userManager.GeneratePasswordResetTokenAsync(user);
        
        // Encode for URL
        var encodedToken = WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(token));
        
        var resetLink = $"{_settings.ClientUrl}/reset-password?email={email}&token={encodedToken}";
        await _email.SendPasswordResetAsync(user.Email!, resetLink);
        
        return Result.Success();
    }
    
    public async Task<Result> ResetAsync(ResetPasswordDto dto)
    {
        var user = await _userManager.FindByEmailAsync(dto.Email);
        if (user == null) return Result.Failure("Invalid request");
        
        // Decode token
        var tokenBytes = WebEncoders.Base64UrlDecode(dto.Token);
        var token = Encoding.UTF8.GetString(tokenBytes);
        
        var result = await _userManager.ResetPasswordAsync(user, token, dto.NewPassword);
        if (!result.Succeeded)
            return Result.Failure(result.Errors.Select(e => e.Description));
        
        // Invalidate all refresh tokens on password change
        await _tokenService.RevokeAllUserTokensAsync(user.Id);
        
        return Result.Success();
    }
}
```

#### 6. **Architecture**: "Design a multi-tenant auth system"

```csharp
// 1. Tenant Resolution Middleware (runs early)
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context, ITenantService tenantService)
    {
        // Strategy 1: Subdomain (tenant1.app.com)
        var host = context.Request.Host.Host;
        var subdomain = host.Split('.').FirstOrDefault();
        
        // Strategy 2: Header (X-Tenant-ID)
        // Strategy 3: JWT Claim (tenant_id)
        // Strategy 4: Path (/tenant1/api/...)
        
        var tenant = await tenantService.GetByIdentifierAsync(subdomain);
        if (tenant == null)
        {
            context.Response.StatusCode = 404;
            return;
        }
        
        context.SetTenant(tenant); // Extension method for HttpContext
        await _next(context);
    }
}

// 2. Tenant-Scoped DbContext
public class TenantDbContext : DbContext
{
    private readonly ITenantContext _tenant;
    
    public TenantDbContext(DbContextOptions opts, ITenantContext tenant) : base(opts)
    {
        _tenant = tenant;
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Global query filter for tenant isolation
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
            {
                modelBuilder.Entity(entityType.ClrType)
                    .HasQueryFilter(e => EF.Property<Guid>(e, "TenantId") == _tenant.TenantId);
            }
        }
    }
}

// 3. Tenant-Aware UserManager
// Each tenant has separate user store or shared with TenantId
public class TenantUserStore : UserStore<AppUser, AppRole, TenantDbContext, Guid>
{
    public TenantUserStore(TenantDbContext context, IdentityErrorDescriber describer) 
        : base(context, describer) { }
    
    public override async Task<AppUser> FindByNameAsync(string normalizedUserName, CancellationToken ct)
    {
        return await Users.FirstOrDefaultAsync(u => 
            u.NormalizedUserName == normalizedUserName && u.TenantId == _tenantContext.TenantId, ct);
    }
}
```

#### 7. **Performance**: "Optimize token validation for high-throughput API"

```csharp
// 1. Use Symmetric Key (HS256) for speed, or RS256 with key caching
// 2. Disable unnecessary validations
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidateAudience = true,
    ValidateLifetime = true,
    ValidateIssuerSigningKey = true,
    ValidIssuer = settings.Issuer,
    ValidAudience = settings.Audience,
    IssuerSigningKey = _cachedSigningKey,  // Pre-computed
    ClockSkew = TimeSpan.Zero,
    RequireExpirationTime = true,
    RequireSignedTokens = true
};

// 3. Cache Signing Keys (for JWKS rotation)
public class CachingKeyProvider : IConfigurationManager<OpenIdConnectConfiguration>
{
    private readonly MemoryCache _cache = new(new MemoryCacheOptions());
    private readonly HttpClient _http;
    private readonly string _jwksUri;
    
    public async Task<OpenIdConnectConfiguration> GetConfigurationAsync(CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync("jwks", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);
            var response = await _http.GetStringAsync(_jwksUri, ct);
            return new OpenIdConnectConfiguration(new JsonWebKeySet(response));
        });
    }
}

// 4. Validate locally, no network calls on hot path
```

#### 8. **Debugging**: "User gets 401 but token looks valid"

**Checklist:**
```csharp
// 1. Check token expiration
var handler = new JwtSecurityTokenHandler();
var jwt = handler.ReadJwtToken(token);
Console.WriteLine($"Exp: {jwt.ValidTo} (UTC), Now: {DateTime.UtcNow}");

// 2. Check clock skew
// TokenValidationParameters.ClockSkew = TimeSpan.Zero;

// 3. Check Issuer/Audience EXACT match (case-sensitive)
// ValidIssuer = "https://auth.example.com"  ≠  "https://auth.example.com/"

// 4. Check signing key matches
// Symmetric: Same secret on auth server and API
// Asymmetric: Public key matches private key

// 5. Check claims mapping
// NameClaimType = "name" vs "sub" vs "email"
// RoleClaimType = "role" vs "roles" vs "http://schemas.microsoft.com/ws/2008/06/identity/claims/role"

// 6. Check middleware order
app.UseAuthentication();  // MUST be before UseAuthorization
app.UseAuthorization();

// 7. Check [Authorize] on correct endpoint
// MapControllers() vs MapGet() - both need auth middleware

// 8. For cookies: SameSite, Secure, Domain settings
options.Cookie.SameSite = SameSiteMode.Lax;  // or None + Secure for cross-site
```

#### 9. **Code**: "Implement API Key authentication alongside JWT"

```csharp
// Scheme: "ApiKey"
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
{
    private readonly IApiKeyValidator _validator;
    
    public ApiKeyAuthenticationHandler(
        IOptionsMonitor<ApiKeyAuthenticationOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock,
        IApiKeyValidator validator) 
        : base(options, logger, encoder, clock)
    {
        _validator = validator;
    }
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-API-Key", out var apiKeyHeader))
            return AuthenticateResult.NoResult();
        
        var apiKey = apiKeyHeader.FirstOrDefault();
        if (string.IsNullOrEmpty(apiKey))
            return AuthenticateResult.NoResult();
        
        var validationResult = await _validator.ValidateAsync(apiKey);
        if (!validationResult.IsValid)
            return AuthenticateResult.Fail("Invalid API Key");
        
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, validationResult.ClientId),
            new(ClaimTypes.Name, validationResult.ClientName),
            new("client_type", "api_key")
        };
        claims.AddRange(validationResult.Scopes.Select(s => new Claim("scope", s)));
        
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);
        
        return AuthenticateResult.Success(ticket);
    }
}

// Registration
builder.Services.AddAuthentication()
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, ...)
    .AddScheme<ApiKeyAuthenticationOptions, ApiKeyAuthenticationHandler>("ApiKey", null);

// Policy accepting either
builder.Services.AddAuthorization(opt =>
{
    opt.DefaultPolicy = new AuthorizationPolicyBuilder()
        .AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme, "ApiKey")
        .RequireAuthenticatedUser()
        .Build();
});
```

#### 10. **Testing**: "Unit test authorization policies"

```csharp
public class AuthorizationTests
{
    private readonly IAuthorizationService _authService;
    
    public AuthorizationTests()
    {
        var services = new ServiceCollection();
        services.AddAuthorization(opt =>
        {
            opt.AddPolicy("AdminOnly", p => p.RequireRole("Admin"));
            opt.AddPolicy("OrderOwner", p => p.Requirements.Add(new OrderOwnerRequirement()));
        });
        services.AddSingleton<IAuthorizationHandler, OrderOwnerHandler>();
        
        var provider = services.BuildServiceProvider();
        _authService = provider.GetRequiredService<IAuthorizationService>();
    }
    
    [Fact]
    public async Task AdminOnlyPolicy_AllowsAdmin()
    {
        var user = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.Role, "Admin")
        }));
        
        var result = await _authService.AuthorizeAsync(user, "AdminOnly");
        Assert.True(result.Succeeded);
    }
    
    [Fact]
    public async Task OrderOwnerPolicy_AllowsOwner()
    {
        var user = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "user-123")
        }));
        
        var order = new Order { CustomerId = Guid.Parse("user-123") };
        
        var result = await _authService.AuthorizeAsync(user, order, "OrderOwner");
        Assert.True(result.Succeeded);
    }
    
    [Fact]
    public async Task OrderOwnerPolicy_DeniesNonOwner()
    {
        var user = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "user-456")
        }));
        
        var order = new Order { CustomerId = Guid.Parse("user-123") };
        
        var result = await _authService.AuthorizeAsync(user, order, "OrderOwner");
        Assert.False(result.Succeeded);
    }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **Token Expiration**: `ClockSkew = TimeSpan.Zero` - no grace period
- [ ] **Refresh Token Rotation**: Always revoke old, issue new - prevents replay
- [ ] **Refresh Token Storage**: Hash in DB (like passwords), not plain text
- [ ] **Cookie vs Header**: SPA = HttpOnly Cookie + CSRF; Mobile = Header
- [ ] **SameSite Cookies**: `Lax` for auth, `None; Secure` for cross-site
- [ ] **Claims Transformation**: Run after auth, before authz for enrichment
- [ ] **Role vs Claim**: Roles are claims (`ClaimTypes.Role`), but claims are more flexible
- [ ] **Policy Evaluation**: ALL requirements must succeed (AND logic)
- [ ] **Resource-Based Auth**: Pass resource to `AuthorizeAsync(user, resource, policy)`
- [ ] **External Login**: Link accounts, don't create duplicates
- [ ] **Password Reset Token**: One-time use, short expiry, invalidate on password change
- [ ] **Email Confirmation**: Required for production, prevent unverified login
- [ ] **Lockout**: Configure `MaxFailedAccessAttempts` and `DefaultLockoutTimeSpan`
- [ ] **Security Stamp**: Updated on password change, role change - invalidates tokens
- [ ] **Token Revocation**: No built-in for JWT - use refresh token revocation + short JWT

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Auth Minimal API**
```csharp
var auth = app.MapGroup("/api/auth").WithTags("Authentication");

auth.MapPost("/register", async (RegisterDto dto, IAuthService svc) =>
{
    var result = await svc.RegisterAsync(dto);
    return result.ToHttpResult();
});

auth.MapPost("/login", async (LoginDto dto, IAuthService svc) =>
{
    var result = await svc.LoginAsync(dto);
    return result.ToHttpResult();
});

auth.MapPost("/refresh", async (RefreshDto dto, IAuthService svc) =>
{
    var result = await svc.RefreshTokenAsync(dto.RefreshToken);
    return result.ToHttpResult();
}).RequireAuthorization(); // Must have valid access token to refresh

auth.MapPost("/logout", [Authorize] async (ClaimsPrincipal user, IAuthService svc) =>
{
    var userId = Guid.Parse(user.FindFirstValue(ClaimTypes.NameIdentifier)!);
    await svc.RevokeAllTokensAsync(userId);
    return Results.Ok();
});

auth.MapPost("/forgot-password", async (ForgotPasswordDto dto, IAuthService svc) =>
{
    await svc.ForgotPasswordAsync(dto.Email);
    return Results.Ok("If account exists, reset email sent");
});

auth.MapPost("/reset-password", async (ResetPasswordDto dto, IAuthService svc) =>
{
    var result = await svc.ResetPasswordAsync(dto);
    return result.ToHttpResult();
});
```

#### 2. **Claims Transformation (Enrichment)**
```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer(options =>
    {
        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = async context =>
            {
                var userId = context.Principal.FindFirstValue(ClaimTypes.NameIdentifier);
                var userService = context.HttpContext.RequestServices.GetRequiredService<IUserService>();
                var permissions = await userService.GetPermissionsAsync(Guid.Parse(userId!));
                
                var claimsIdentity = (ClaimsIdentity)context.Principal.Identity;
                foreach (var perm in permissions)
                    claimsIdentity.AddClaim(new Claim("permission", perm));
            }
        };
    });

// Or use IClaimsTransformation (runs on EVERY request)
public class PermissionClaimsTransformer : IClaimsTransformation
{
    private readonly IUserService _userService;
    
    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        if (principal.Identity?.IsAuthenticated != true) return principal;
        
        var userId = principal.FindFirstValue(ClaimTypes.NameIdentifier);
        if (string.IsNullOrEmpty(userId)) return principal;
        
        // Check if already transformed (avoid duplicate on refresh)
        if (principal.HasClaim("permissions_loaded", "true")) return principal;
        
        var permissions = await _userService.GetPermissionsAsync(Guid.Parse(userId));
        var identity = (ClaimsIdentity)principal.Identity;
        
        foreach (var perm in permissions)
            identity.AddClaim(new Claim("permission", perm));
        
        identity.AddClaim(new Claim("permissions_loaded", "true"));
        return principal;
    }
}

builder.Services.AddScoped<IClaimsTransformation, PermissionClaimsTransformer>();
```

#### 3. **Secure Cookie Config for SPA**
```csharp
builder.Services.AddAuthentication()
    .AddCookie("AuthCookie", options =>
    {
        options.Cookie.Name = "__Host-auth";  // __Host- prefix = secure only
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Strict;  // CSRF protection
        options.Cookie.MaxAge = TimeSpan.FromDays(30);
        options.SlidingExpiration = true;
        options.ExpireTimeSpan = TimeSpan.FromDays(30);
        
        options.Events.OnRedirectToLogin = context =>
        {
            context.Response.StatusCode = 401;
            return Task.CompletedTask;
        };
    });

// Login sets cookie
await HttpContext.SignInAsync("AuthCookie", principal, new AuthenticationProperties
{
    IsPersistent = true,
    ExpiresUtc = DateTimeOffset.UtcNow.AddDays(30)
});

// Logout clears cookie
await HttpContext.SignOutAsync("AuthCookie");
```

---

### PRACTICE PROBLEMS

1. **Implement**: Complete auth system with: Register, Login, Refresh, Logout, Forgot/Reset Password, Email Confirmation, MFA
2. **Design**: Auth for microservices - shared IdentityServer vs per-service JWT validation
3. **Security**: Add rate limiting to auth endpoints (login: 5/min, register: 3/hour)
4. **Debug**: Token works in Postman but not browser - diagnose CORS/cookie issues
5. **Scale**: Design token validation for 100K req/sec - caching, key rotation, monitoring

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| AuthN vs AuthZ | AuthN = Who (identity). AuthZ = What (permissions). Order: AuthN → AuthZ |
| JWT Structure | Header.Payload.Signature (Base64Url encoded, dot-separated) |
| JWT Validation | Signature → Expiry → Issuer → Audience → Claims |
| Refresh Token Rotation | Issue new refresh token, revoke old one. Prevents replay attacks |
| Access Token Lifetime | Short (15-30 min). Long = no revocation |
| Policy vs Role | Role = simple claim. Policy = complex logic (requirements + handlers) |
| Resource-Based Auth | `AuthorizeAsync(user, resource, policy)` - passes resource to handler |
| Cookie Security | HttpOnly + Secure + SameSite=Strict/Lax + __Host- prefix |
| ClaimsTransformation | Runs per request. Add claims from DB. Cache results! |
| Security Stamp | Invalidates tokens on password/role change. Updated by UserManager |

---

## NEXT TOPIC: `06-CACHING.md`

> **Study Tip**: Build a working auth API with: JWT + Refresh rotation, Email confirmation, Password reset, Policy-based auth (OrderOwner), MFA enable/verify. Test with Postman/Swagger.