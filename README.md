# batchReference

import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.batch.core.job.builder.JobBuilderFactory;
import org.springframework.batch.core.step.builder.StepBuilderFactory;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.TaskExecutor;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final TaskExecutor taskExecutor;

    public BatchConfig(JobBuilderFactory jobBuilderFactory,
                       StepBuilderFactory stepBuilderFactory,
                       TaskExecutor taskExecutor) {
        this.jobBuilderFactory = jobBuilderFactory;
        this.stepBuilderFactory = stepBuilderFactory;
        this.taskExecutor = taskExecutor;
    }

    @Bean
    public Step step1(ItemReader<String> reader,
                      ItemProcessor<String, String> processor,
                      ItemWriter<String> writer) {
        return stepBuilderFactory.get("step1")
                .<String, String>chunk(10)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .taskExecutor(taskExecutor) // Assign TaskExecutor here
                .build();
    }
}



++++++++++++++++++++++++++


@Configuration
public class FileToJobIntegrationConfig {

    @Bean
    public IntegrationFlow fileReadingFlow() {
        return IntegrationFlows
            .from(Files.inboundAdapter(new File("input-directory"))
                .patternFilter("*.txt"), e -> e.poller(Pollers.fixedDelay(1000)))
            .channel("inputChannel")
            .get();
    }

    @Transformer(inputChannel = "inputChannel", outputChannel = "jobRequestChannel")
    public JobLaunchRequest toRequest(Message<File> message) {
        File file = message.getPayload();
        JobParametersBuilder jobParametersBuilder = new JobParametersBuilder();
        jobParametersBuilder.addString("input.file.name", file.getAbsolutePath());
        jobParametersBuilder.addLong("file.timestamp", file.lastModified());

        return new JobLaunchRequest(myJob, jobParametersBuilder.toJobParameters());
    }

    @Bean
    public IntegrationFlow jobLaunchingFlow(JobLaunchingGateway jobLaunchingGateway) {
        return IntegrationFlows
            .from("jobRequestChannel")
            .handle(jobLaunchingGateway)
            .get();
    }

    @MessagingGateway
    public interface JobLaunchingGateway {
        @Gateway(requestChannel = "jobRequestChannel")
        void launch(JobLaunchRequest request);
    }
}

