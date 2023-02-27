# 电商app用户钱包
业务场景：电商业务中，需要给电商app设计一个用户钱包，用户可以往钱包中充值，购买商品时用户可以使用钱包中的钱消费，商品申请退款成功后钱会退回钱包中，用户也可以申请提现把钱提到银行卡中。

# 第一，数据库创建
## 用户钱包表
CREATE TABLE `user_wallet` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_id` bigint(20) unsigned NOT NULL COMMENT '用户id',
  `balance` decimal(18,2) NOT NULL DEFAULT '0.00' COMMENT '余额',
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户钱包表';

## 钱包变动明细表
CREATE TABLE `wallet_transaction` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_id` bigint(20) unsigned NOT NULL COMMENT '用户id',
  `transaction_type` varchar(20) NOT NULL COMMENT '交易类型：RECHARGE-充值，PURCHASE-购买，REFUND-退款，WITHDRAWAL-提现',
  `amount` decimal(18,2) NOT NULL COMMENT '交易金额',
  `balance` decimal(18,2) NOT NULL COMMENT '交易后余额',
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_id_created_at` (`user_id`,`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='钱包变动明细表';

//其中 user_wallet 表用于存储用户钱包余额信息，wallet_transaction 表用于存储用户钱包的交易明细，包括充值、消费、退款、提现四种交易类型。

# 第二，API接口实现
## UserWalletController
@RestController
@RequestMapping("/wallet")
public class UserWalletController {

    @Autowired
    private UserWalletService userWalletService;

    /**
     * 查询用户钱包余额
     *
     * @param userId 用户ID
     * @return 用户钱包余额
     */
    @GetMapping("/balance/{userId}")
    public BigDecimal getBalance(@PathVariable("userId") Integer userId) {
        return userWalletService.getBalance(userId);
    }

    /**
     * 用户消费接口
     *
     * @param userId 用户ID
     * @param amount 消费金额
     * @return 操作结果
     */
    @PostMapping("/consume/{userId}")
    public String consume(@PathVariable("userId") Integer userId, @RequestParam("amount") BigDecimal amount) {
        try {
            userWalletService.consume(userId, amount);
            return "操作成功";
        } catch (Exception e) {
            return "操作失败：" + e.getMessage();
        }
    }

    /**
     * 用户退款接口
     *
     * @param userId 用户ID
     * @param amount 退款金额
     * @return 操作结果
     */
    @PostMapping("/refund/{userId}")
    public String refund(@PathVariable("userId") Integer userId, @RequestParam("amount") BigDecimal amount) {
        try {
            userWalletService.refund(userId, amount);
            return "操作成功";
        } catch (Exception e) {
            return "操作失败：" + e.getMessage();
        }
    }

    /**
     * 查询用户钱包金额变动明细的接口
     *
     * @param userId 用户ID
     * @return 用户钱包金额变动明细
     */
    @GetMapping("/transactions/{userId}")
    public List<WalletTransaction> getTransactions(@PathVariable("userId") Integer userId) {
        return userWalletService.getTransactions(userId);
    }
}


# 第三，Service实现
## UserWalletService
public interface UserWalletService {

    /**
     * 查询用户钱包余额
     *
     * @param userId 用户ID
     * @return 用户钱包余额
     */
    BigDecimal getBalance(Integer userId);

    /**
     * 用户消费
     *
     * @param userId 用户ID
     * @param amount 消费金额
     */
    void consume(Integer userId, BigDecimal amount);
    
    /**
     * 用户退款
     *
     * @param userId
    * @param amount 退款金额
     */
    void refund(Integer userId, BigDecimal amount);

    /**
     * 查询用户钱包金额变动明细
     *
     * @param userId 用户ID
     * @return 用户钱包金额变动明细
     */
    List<WalletTransaction> getTransactions(Integer userId);
}

@Service
public class UserWalletServiceImpl implements UserWalletService {
    @Autowired
    private UserWalletDao userWalletDao;

    @Autowired
    private WalletTransactionDao walletTransactionDao;

    @Override
    public BigDecimal getBalance(Integer userId) {
        UserWallet userWallet = userWalletDao.findByUserId(userId);
        return userWallet == null ? BigDecimal.ZERO : userWallet.getBalance();
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void consume(Integer userId, BigDecimal amount) {
        UserWallet userWallet = userWalletDao.findByUserId(userId);
        if (userWallet == null) {
            throw new RuntimeException("用户钱包不存在");
        }
        BigDecimal balance = userWallet.getBalance();
        if (balance.compareTo(amount) < 0) {
            throw new RuntimeException("余额不足");
        }
        userWallet.setBalance(balance.subtract(amount));
        userWalletDao.save(userWallet);

        WalletTransaction transaction = new WalletTransaction();
        transaction.setUserId(userId);
        transaction.setTransactionType("consumption");
        transaction.setAmount(amount.negate());
        walletTransactionDao.save(transaction);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void refund(Integer userId, BigDecimal amount) {
        UserWallet userWallet = userWalletDao.findByUserId(userId);
        if (userWallet == null) {
            throw new RuntimeException("用户钱包不存在");
        }
        BigDecimal balance = userWallet.getBalance();
        userWallet.setBalance(balance.add(amount));
        userWalletDao.save(userWallet);

        WalletTransaction transaction = new WalletTransaction();
        transaction.setUserId(userId);
        transaction.setTransactionType("refund");
        transaction.setAmount(amount);
        walletTransactionDao.save(transaction);
    }

    @Override
    public List<WalletTransaction> getTransactions(Integer userId) {
        return walletTransactionDao.findByUserId(userId);
    }
}

# 第四，DAO和实体类的实现
## UserWallet实体类
@Entity
@Table(name = "user_wallet")
public class UserWallet {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "balance", nullable = false)
    private BigDecimal balance;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    // getters and setters
}

## WalletTransaction实体类
@Entity
@Table(name = "wallet_transaction")
public class WalletTransaction {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "transaction_type", nullable = false)
    @Enumerated(EnumType.STRING)
    private TransactionType transactionType;

    @Column(name = "amount", nullable = false)
    private BigDecimal amount;

    @Column(name = "balance", nullable = false)
    private BigDecimal balance;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    // constructors, getters and setters
}

