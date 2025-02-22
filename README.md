Thiết Kế Service Lập Lịch mt-job trong Môi Trường Microservice
1. Lý do cần tách biệt service lập lịch
Trong hệ thống, các tác vụ nền (background jobs) đóng vai trò quan trọng, chẳng hạn như:
    • Tự động xử lý dữ liệu định kỳ
    • Gửi thông báo theo lịch trình
Ban đầu, các tác vụ này thường được tích hợp trực tiếp vào service chính. Tuy nhiên, khi hệ thống mở rộng, việc tách riêng một service chuyên trách cho lập lịch (mt-job) là cần thiết để:
    • Giảm tải cho service chính
    • Dễ dàng quản lý, cấu hình và mở rộng
    • Hỗ trợ tốt hơn trong môi trường microservice phân tán
    • Bất lợi của việc tách service như này là code nhiều nhưng sẽ tốt cho sau này mở rộng lớn
    • 
2. Lựa chọn thư viện lập lịch: Quartz vs Spring Scheduler
Thay vì sử dụng Spring Boot Scheduler, ta chọn Quartz vì:
 Hỗ trợ mạnh mẽ cho các job động, có thể cấu hình phức tạp
 Quản lý job tốt hơn trong môi trường microservice
 Dễ dàng tích hợp với các hệ thống phân tán
Bước đầu tiên là xây dựng JobCommonConfig – một class chung giúp chuẩn hóa các job và cung cấp các tính năng mở rộng.
Ở đây là 1 config chung còn việc Muốn config cho Job thực thi các công việc cụ thể thì có thể implement class này:
public abstract class JobConfig implements Job {
    private static final Logger logger = LoggerFactory.getLogger(JobConfig.class);
    @Autowired
    protected RedisLockService redisLockService;
    @Autowired
    private Scheduler scheduler;
    @Autowired
    private JobHistoryService jobHistoryService;

    @Value("${monitor.redis.lock.default.timeout:10000}")
    private Long lockTimeoutMs;

    @Override
    public void execute(JobExecutionContext jobExecutionContext) {
        //save log db
        JobHistoryEntity jobHistoryEntity = new JobHistoryEntity();
        jobHistoryEntity.setStartDate(new Date());
        var jobDetail = jobExecutionContext.getJobDetail();
        String keyLock =
                String.format("%s_%s1", jobDetail.getKey().getGroup(), jobDetail.getKey().getName());
        if (redisLockService.tryLock(keyLock, getLockTimeoutMs())) {
            try {
                this.executeJob(jobExecutionContext);
            } catch (BusinessException
                     |
                     SchedulerException
                     |
                     RuntimeException e) {
                jobHistoryEntity.setEndDate(new Date());
                jobHistoryEntity.setJobId(NumberUtils.toLong(jobDetail.getKey().getName(), 0L));
                jobHistoryEntity.setJobType(getGroupName());
                jobHistoryEntity.setDescription(
                        ExceptionUtils.getMessage(e)
                                +
                                ExceptionUtils.getRootCauseMessage(e)
                                +
                                ExceptionUtils.getStackTrace(e));
                jobHistoryService.save(jobHistoryEntity);
                logger.error("Job excute error", e);
            } catch (IOException e) {
                throw new RuntimeException(e);
            } finally {
                redisLockService.unlock(keyLock);
            }
        }
    }

    /**
     * function job.
     *
     * @param context JobExecutionContext.
     */
    public abstract void executeJob(JobExecutionContext context)
            throws BusinessException, SchedulerException, BusinessRuntimeException, IOException;

    /**
     * function schedule job if not exists.
     *
     * @param intervalTime time interval.
     * @param key          name of job.
     */
    public void scheduleJobIfNotExists(String key, Long intervalTime)
            throws SchedulerException {
        JobKey jobKey = JobKey.jobKey(key, getGroupName());

        if (scheduler.checkExists(jobKey)) {
            logger.info("Job already exists with job name: " + key + ", job group: " + getGroupName());
            return;
        }
        logger.info("Creating a new job with job name: " + key + ", job group: " + getGroupName());

        JobDetail jobDetail = JobBuilder.newJob(this.getClass())
                .withIdentity(jobKey)
                .build();

        Trigger trigger = buildTrigger(key, intervalTime);
        scheduler.scheduleJob(jobDetail, trigger);
        logger.info(
                "Job scheduled successfully with job name: " + key + ", job group: " + getGroupName());
    }

    /**
     * function build trigger.
     *
     * @param key          name of job.
     * @param intervalTime time interval.
     * @return trigger object.
     */
    public Trigger buildTrigger(String key, Long intervalTime) {
        //since Wed Nov 20 00:00:00 UTC 2024
        Calendar startAt = Calendar.getInstance();
        startAt.set(Calendar.YEAR, 2024);
        Date startTime = startAt.getTime();
        return TriggerBuilder.newTrigger()
                .withIdentity(key + "Trigger", getGroupName())
                .startAt(startTime)
                .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                        .withIntervalInMilliseconds(intervalTime)
                        .repeatForever())
                .build();
    }

    /**
     * function reschedule job.
     *
     * @param key          name of job.
     * @param intervalTime time interval.
     */
    public void rescheduleJob(String key, Long intervalTime)
            throws SchedulerException {
        removeJob(key);
        scheduleJobIfNotExists(key, intervalTime);
    }

    /**
     * Kafka consumer.
     *
     * @param data data
     */
    public void kafkaConsumer(KafkaJobModel data) {
        //nothing Override logic
    }

    /**
     * function remove job.
     *
     * @param key name of job.
     */
    public void removeJob(String key) throws SchedulerException {
        JobKey jobKey = new JobKey(key, getGroupName());
        scheduler.deleteJob(jobKey);
        logger.info("Job was removed successfully job name: " + key + ", job group: " + getGroupName());
    }

    /**
     * Get lock redis timeout.
     *
     * @return config timeout
     */
    public Long getLockTimeoutMs() {
        return lockTimeoutMs;
    }

    /**
     * Init job when start app.
     *
     * @throws SchedulerException ex
     */
    @PostConstruct
    public void initScheduler() throws SchedulerException {
        boolean isGroupExist = scheduler.getJobGroupNames().contains(getGroupName());
        if (isGroupExist || CollectionUtils.isEmpty(getListConfig())) {
            return;
        }
        for (Map.Entry<String, Long> entry : getListConfig().entrySet()) {
            scheduleJobIfNotExists(entry.getKey(), entry.getValue());
        }
        logger.info("Scheduler initialized successfully.");
    }

    /**
     * List config with key = keyJob and value = interval time.
     *
     * @return Map
     */
    public abstract Map<String, Long> getListConfig();

    /**
     * Config group name of job.
     *
     * @return group name
     */
    public abstract String getGroupName();

    /**
     * get redis lock key.
     *
     * @param jobKey jobKey
     * @return lock redis key
     */
    private String getKeyLockRedis(String jobKey) {
        return String.format("%s_%s", getGroupName(), jobKey);
    }

}




3. Vấn đề chạy job đồng thời trong môi trường microservice
Trong môi trường microservice, giả sử có nhiều instance của mt-job chạy trên cùng một node hoặc nhiều node khác nhau. Khi đó, có hai vấn đề lớn có thể xảy ra:
Trường hợp 1: Hai job chạy lệch thời gian nhau
 Giải pháp:
Có thể cấu hình một mốc thời gian cố định để tất cả instance mt-job chạy cùng một thời điểm. Quartz hỗ trợ tính năng này, giúp đồng bộ thời gian chạy job trên toàn hệ thống.
 Trường hợp 2: Hai job chạy cùng thời điểm
Đây là vấn đề nghiêm trọng, đặc biệt với các hệ thống tài chính, ví dụ:
Nếu một job tính lãi suất chạy hai lần liên tiếp, khách hàng có thể nhận gấp đôi tiền lãi – hậu quả rất lớn!
 Giải pháp phổ biến:
1️⃣ Dùng Oracle DB làm cơ chế khóa (Locking Mechanism)
    • Lưu một khóa (key) trong DB, service nào lấy được khóa sẽ chạy job.
    • Nếu khóa đã bị chiếm, job khác sẽ không chạy.
    •  Nhược điểm: Nếu service gặp lỗi hoặc dừng đột ngột, khóa có thể bị "mắc kẹt" vĩnh viễn và không được giải phóng.
2️⃣ Dùng Redis làm khóa phân tán
    • Redis là lựa chọn tối ưu, vì có thể thiết lập TTL (time-to-live) cho khóa để tránh tình trạng "mắc kẹt".
    • Có hai cách triển khai:
        ◦ Sử dụng Redisson: Một thư viện hỗ trợ khóa phân tán, tự động xử lý giải phóng khóa khi có sự cố.
        ◦ Dùng Redis thuần (native Redis): Sử dụng lệnh SETNX (Set if Not Exists) đảm bảo chỉ một service có thể chiếm khóa tại một thời điểm.
 Lưu ý sai lầm thường gặp:
Nhiều người dùng Redis mà chỉ sử dụng SET và GET, nghĩ rằng nếu có key rồi thì service khác không thể set được. Thực tế không phải vậy! Chỉ có SETNX mới giúp đảm bảo việc lock an toàn.
Code cơ chế khóa nhé :

@Service
public class RedisLockServiceImpl {

    @Autowired
    private StringRedisTemplate redisTemplate;


    public String tryLock(String lockKey, long expireTime) {

        String lockValue = UUID.randomUUID().toString();
        Boolean success = redisTemplate.opsForValue().setIfAbsent(lockKey, “”, expireTime, TimeUnit.SECONDS);
        if (success != null && success) {
            return lockValue;
        }
        return null;
    }


    public void unlock(String lockKey, String lockValue) {
        String currentValue = redisTemplate.opsForValue().get(lockKey);
        if (lockValue.equals(currentValue)) {
            redisTemplate.delete(lockKey);
        }
    }
}


4. Vấn đề khi triển khai Redis trong môi trường Pre-Live
Sau khi hoàn thành UAT (User Acceptance Testing), hệ thống hoạt động trơn tru vì chỉ sử dụng một instance Redis. Tuy nhiên, khi triển khai lên môi trường Pre-Live, nơi hệ thống đã được scale lên nhiều instance Redis (thay vì một), một vấn đề nghiêm trọng xuất hiện:
 Vấn đề:
    • Hai instance Redis hoạt động độc lập thay vì cluster.
    • Khi một service mt-job set khóa ở Redis A, service mt-job khác vẫn có thể set khóa trên Redis B, dẫn đến hai job chạy trùng nhau.
 Giải pháp:
    • Cấu hình Redis thành một cụm (cluster) để đảm bảo dữ liệu khóa được đồng bộ giữa các instance.
    • Nếu không thể dùng cluster, cần thiết lập một cơ chế kiểm tra khóa trên cả hai Redis (có thể kết hợp Redisson để xử lý).
 Bài học rút ra: Khi thiết kế hệ thống, luôn phải tính đến việc scale lên nhiều instance và môi trường triển khai thực tế.

5. Giao tiếp giữa mt-job và service main
Một vấn đề quan trọng khác là làm thế nào để cập nhật hoặc xóa job khi có thay đổi từ service main.
 Cách giao tiếp giữa hai service
Có nhiều phương thức có thể sử dụng:
    • Gọi API trực tiếp (REST, gRPC)
    • Dùng message queue như Kafka, RabbitMQ, ActiveMQ
    • Sử dụng cơ chế event-driven với Redis Pub/Sub hoặc WebSocket
 Lý do chọn Kafka
Trong dự án này, chúng ta sử dụng Kafka thay vì các giải pháp khác. Lý do:
 Hiệu suất cao – Performance rất khó cải thiện, trong khi tính đảm bảo có thể tăng bằng cách điều chỉnh Kafka nhưng ngược lại thì khó hơn rất nhiều( Câu này nghe qua ytb: Tip Javascript chia sẻ nè).
 Hỗ trợ phân tán tốt, phù hợp với kiến trúc microservice.
 Bảo đảm tin cậy (có thể cấu hình để đảm bảo message không bị mất).
Hệ thống sẽ gồm 3 thành phần chính:
1️⃣ Producer (Service Main)
    • Gửi message khi có thay đổi về config job (tạo, cập nhật, xóa).
2️⃣ Broker (Kafka Cluster- Do đội hạ tầng cấu hình và config sẵn cho mình rùi)
    • Là trung gian truyền tải message từ producer đến consumer.
    • Giữ message để tránh mất mát dữ liệu.
3️⃣ Consumer (Service mt-job)
    • Nhận message từ Kafka Broker.
    • Cập nhật lại các job tương ứng.
Kết quả: Khi service main thay đổi cấu hình job, Kafka sẽ đảm bảo mt-job cập nhật đúng theo thời gian thực, giúp hệ thống luôn đồng bộ.
Trong việc này vẽ vời và khen là thế nhưng cũng cần đánh giá năng lực và khả năng cung cấp giải pháp từ dự án như nào đừng chỉ nghe người ta khen mà dùng mà hãy xem có phù hợp không nhé=))
Vấn đề gặp phải khi dùng Kafka
 Đây là một vấn đề thường gặp trong hệ thống Kafka khi triển khai nhiều instance của cùng một service tiêu thụ message. Bản chất của Kafka Consumer Groups chính là:
    • Nếu các consumer thuộc cùng một group, Kafka sẽ chia các partition của topic giữa các consumer đó. Một partition chỉ được xử lý bởi một consumer trong group tại một thời điểm.
    • Nếu mỗi consumer có group riêng, tất cả consumer đều nhận được cùng một message.
 Vấn đề gặp phải
    • Khi triển khai 2 instance của mt-job, chỉ 1 instance nhận message, còn instance kia không nhận được gì.
    • Lý do: Cả hai instance đều thuộc cùng một group-id, nên Kafka chỉ cho một trong số chúng xử lý message.
 Giải pháp khắc phục
    1. Tạo group-id ngẫu nhiên cho mỗi instance
        ◦ Điều này giúp mỗi instance hoạt động như một consumer độc lập, đảm bảo tất cả nhận được message.
        ◦ Thực hiện bằng cách generate group-id động khi khởi động service.
    2. Cập nhật KafkaJobBase để generate group-id động
        ◦ Đoạn code đã có generateGroupId() rồi, nhưng có thể kiểm tra lại như sau:

       @KafkaListener(
 topics = "#{kafkaProperties.topicJob}",
 groupId = "#{T(vn.com.demo.job.configs.kafka.KafkaJobBase).generateGroupId()}"
 )
 public void listen(ConsumerRecord<String, String> record) {
 super.process(record);
 }

 public static String generateGroupId() {
 return \"group-\" + UUID.randomUUID().toString();
 }
       
 Lưu ý quan trọng
    • Việc tạo group-id ngẫu nhiên hi sinh tính tải cao của Kafka (vì mất đi cơ chế load balancing giữa các consumer cùng group).
    • Nhưng bù lại, nó đảm bảo mỗi instance đều nhận đủ message.
    • Đây không phải lỗi công nghệ, mà là một quyết định thiết kế tùy vào yêu cầu hệ thống.
 Khi nào nên dùng cách này?
    • Khi cần mỗi instance xử lý đầy đủ tất cả các message (thay vì chia sẻ chúng).
    • Khi Kafka topic có ít partition nhưng cần scale nhiều consumer (tránh trường hợp consumer rảnh rỗi không có việc làm).
    • Khi cần đảm bảo toàn bộ dữ liệu đến được tất cả instance.
@Service
public abstract class AbstractKafkaConsumerService<K, V> {
    protected abstract Logger logger();

    protected abstract void handleRecord(ConsumerRecord<K, V> record) throws Exception;

    protected boolean isValid(ConsumerRecord<K, V> record) {
        return record != null;
    }

    public void process(ConsumerRecord<K, V> record) {
        try {
            if (isValid(record)) {
                handleRecord(record);
            } else {
                logger().warn("Invalid message received: {}", record);
            }
        } catch (Exception e) {
            logger().error("Error processing message: {}", e.getMessage(), e);
        }
    }
}












@Service
@Component
@EnableKafka
public class KafkaJobBase extends AbstractKafkaConsumerService<String, String> {
    @Autowired
    private ApplicationContext applicationContext;
    static final Logger LOGGER = LoggerFactory.getLogger(KafkaJobBase.class);

    @Override
    protected Logger logger() {
        return LOGGER;
    }

    @Override
    protected void handleRecord(ConsumerRecord<String, String> record) throws Exception {
        if (applicationContext.containsBean(record.key())) {
            var bean = applicationContext.getBean(record.key());
            if (bean instanceof JobConfig) {
                var data = KanbanCommonUtil.stringToBean(record.value(), KafkaJobModel.class);
                JobConfig job = (JobConfig) bean;
                job.kafkaConsumer(data);
            }
        }
    }

    @KafkaListener(
            topics = "#{kafkaProperties.topicJob}",
            groupId = "#{T(vn.com.demo.job.configs.kafka.KafkaJobBase).generateGroupId()}"
    )
    public void listen(ConsumerRecord<String, String> record) {
        super.process(record);
    }

    public static String generateGroupId() {
        return \"group-\" + UUID.randomUUID().toString();
    }
}




6. Tối Ưu Hóa Đa Luồng cho Job Email
Với một số job, ví dụ như job gửi email phức tạp, việc sử dụng đa luồng có thể giúp tăng hiệu năng xử lý song song. Tuy nhiên, cần lưu ý:
    • Đánh giá chi phí khởi tạo và duy trì threadpool: Không phải lúc nào việc khởi tạo đa luồng cũng mang lại hiệu năng cao nếu chi phí khởi tạo và quản lý threadpool vượt quá lợi ích song song hóa.
    • Cấu hình threadpool hợp lý: Sử dụng cấu hình threadpool cho phép điều chỉnh số lượng luồng dựa trên tài nguyên hệ thống và đặc điểm công việc.
    • Tối ưu hóa cho từng job: Trước khi triển khai, cần tính toán xem job có thực sự cần đa luồng hay không để tránh việc khởi tạo vô lý, gây lãng phí tài nguyên.
Cấu hình ThreadPool:

@Configuration
public class ThreadPoolConfig {

    @Value("${thread-pool.pool-size.core}")
    private Integer corePoolSize;

    @Value("${thread-pool.pool-size.max}")
    private Integer maxPoolSize;

    @Value("${thread-pool.queue-capacity}")
    private Integer queueCapacity;

    @Value("${thread-pool.thread-name-prefix}")
    private String threadNamePrefix;

    @Value("${thread-pool.await-termination-seconds}")
    private int awaitTerminationSeconds;

    @Bean(name = "lightWeighTaskExecutor")
    public TaskExecutor lightWeighTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setDaemon(true);
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setAwaitTerminationSeconds(awaitTerminationSeconds);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setThreadNamePrefix(threadNamePrefix);
        executor.initialize();
        return executor;
    }
}
