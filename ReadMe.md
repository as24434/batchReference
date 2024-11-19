Spring Boot 3.3.3 - File Triggered Batch Processing with Shadow Mode
Overview
This project demonstrates a Spring Boot application integrating Spring Batch and Spring Integration. The application processes files based on a trigger mechanism and a CSV file with 10,000 records, each containing 55 fields. Key features include:

Trigger Mechanism: Monitors a directory for trigger files and initiates batch processing.
Batch Job: Processes a CSV file and maps each record to a POJO with 55 fields.
Shadow Mode: Logs the records without processing them when enabled.
Integration Flow: Delays, deletes the trigger file, and launches the batch job.
Project Flow
Sequence Diagram

Detailed Logic with Comments
1. BatchConfig.java
Defines the Spring Batch job and its components.

java
Copy code
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public Job processJob(JobBuilderFactory jobBuilderFactory, Step step1) {
        // Defines the batch job, starting with step1
        return jobBuilderFactory.get("processJob")
                .start(step1)
                .build();
    }

    @Bean
    public Step step1(StepBuilderFactory stepBuilderFactory, ItemReader<Record> reader,
                      ItemProcessor<Record, Record> processor, ItemWriter<Record> writer) {
        // Defines a step that processes records in chunks of 1000
        return stepBuilderFactory.get("step1")
                .<Record, Record>chunk(1000)
                .reader(reader)      // Reads records from CSV
                .processor(processor) // Processes each record
                .writer(writer)       // Writes processed records
                .build();
    }

    @Bean
    public FlatFileItemReader<Record> reader() {
        // Configures a reader to parse CSV and map to Record POJO
        FlatFileItemReader<Record> reader = new FlatFileItemReader<>();
        reader.setResource(new FileSystemResource("input/records.csv"));
        reader.setLinesToSkip(1); // Skip header row
        reader.setLineMapper(new DefaultLineMapper<>() {{
            setLineTokenizer(new DelimitedLineTokenizer() {{
                setDelimiter(",");
                setNames("field1", "field2", /* ... */, "field55"); // Map fields
            }});
            setFieldSetMapper(fieldSet -> {
                Record record = new Record();
                record.setField1(fieldSet.readString("field1"));
                record.setField2(fieldSet.readString("field2"));
                // ... map all 55 fields
                return record;
            });
        }});
        return reader;
    }

    @Bean
    public ItemProcessor<Record, Record> processor() {
        // Example processor: add validation or transformation logic
        return item -> {
            if (item.getField1().isEmpty()) {
                throw new IllegalArgumentException("Field1 cannot be empty");
            }
            // Add more transformation logic if needed
            return item;
        };
    }

    @Bean
    public ItemWriter<Record> writer() {
        // Writes the processed records to the console
        return items -> items.forEach(item -> System.out.println("Processed: " + item));
    }
}
2. IntegrationConfig.java
Handles the trigger file processing and launches the batch job.

java
Copy code
@Configuration
@IntegrationComponentScan
public class IntegrationConfig {

    @Bean
    public FileReadingMessageSource fileReader() {
        // Monitors the 'input/triggers' directory for new files
        FileReadingMessageSource source = new FileReadingMessageSource();
        source.setDirectory(new File("input/triggers"));
        source.setScanEachPoll(true);
        source.setAutoCreateDirectory(true);
        return source;
    }

    @Bean
    public FileWritingMessageHandler fileDeleter() {
        // Deletes the trigger file after it is read
        FileWritingMessageHandler handler = new FileWritingMessageHandler(new File("processed/triggers"));
        handler.setDeleteSourceFiles(true);
        return handler;
    }

    @Bean
    public IntegrationFlow integrationFlow(JobLauncher jobLauncher, Job processJob) {
        // Defines the flow:
        // 1. Reads trigger files
        // 2. Delays for 5 seconds
        // 3. Deletes trigger files
        // 4. Launches the batch job
        return IntegrationFlows.from(fileReader(), spec -> spec.poller(p -> p.fixedDelay(5000)))
                .handle(fileDeleter()) // Deletes the trigger file
                .transform(File.class, file -> new JobLaunchRequest(processJob, null))
                .handle(new JobLaunchingMessageHandler(jobLauncher)) // Launches the job
                .get();
    }
}
3. ShadowModeConfig.java
Manages shadow mode functionality.

java
Copy code
@Configuration
public class ShadowModeConfig {

    @Value("${shadow.mode:false}")
    private boolean shadowMode;

    @Bean
    public ItemWriter<Record> writer() {
        return items -> {
            if (shadowMode) {
                // Logs records instead of processing in shadow mode
                items.forEach(item -> System.out.println("Shadow mode: " + item));
            } else {
                // Actual processing logic
                items.forEach(item -> System.out.println("Processed: " + item));
            }
        };
    }
}
4. Record.java
POJO representing a single record with 55 fields.

java
Copy code
public class Record {

    private String field1;
    private String field2;
    // ...
    private String field55;

    // Getters and Setters
    public String getField1() { return field1; }
    public void setField1(String field1) { this.field1 = field1; }
    // ... repeat for all fields

    @Override
    public String toString() {
        return "Record{" +
                "field1='" + field1 + '\'' +
                ", field2='" + field2 + '\'' +
                // ...
                ", field55='" + field55 + '\'' +
                '}';
    }
}
5. JobLauncherService.java
Service to programmatically launch batch jobs.

java
Copy code
@Service
public class JobLauncherService {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job job;

    public void launchJob() {
        try {
            JobParameters params = new JobParametersBuilder()
                    .addLong("time", System.currentTimeMillis())
                    .toJobParameters();
            jobLauncher.run(job, params);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
6. FileTriggerGateway.java
Integration gateway to trigger flows.

java
Copy code
@MessagingGateway
public interface FileTriggerGateway {
    void trigger(String filePath);
}
7. JobController.java
REST API to manually trigger the batch job.

java
Copy code
@RestController
@RequestMapping("/job")
public class JobController {

    @Autowired
    private JobLauncherService jobLauncherService;

    @PostMapping("/trigger")
    public ResponseEntity<String> triggerJob() {
        jobLauncherService.launchJob();
        return ResponseEntity.ok("Job triggered successfully!");
    }
}
Usage
Start Application:

bash
Copy code
./gradlew bootRun
Add Trigger File:
Place a file in the input/triggers directory.

Observe Processing:
Records from records.csv are processed and logged.

Shadow Mode:
Enable by setting shadow.mode=true in application.properties.

