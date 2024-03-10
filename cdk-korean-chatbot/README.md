# CDK로 인프라 구현하기 

여기서는 Typescript를 이용하여 한국어 Chatbot을 위한 인프라를 설치합니다.

Multi-Region LLM을 위해 LLM profile을 정의합니다.

```typescript
const claude_instant = [
  {
    "bedrock_region": "us-west-2", // Oregon
    "model_type": "claude",
    "model_id": "anthropic.claude-instant-v1",
    "maxOutputTokens": "8196"
  },
  {
    "bedrock_region": "us-east-1", // N.Virginia
    "model_type": "claude",
    "model_id": "anthropic.claude-instant-v1",
    "maxOutputTokens": "8196"
  },
  {
    "bedrock_region": "ap-northeast-1", // Tokyo
    "model_type": "claude",
    "model_id": "anthropic.claude-instant-v1",
    "maxOutputTokens": "8196"
  },    
  {
    "bedrock_region": "eu-central-1", // Europe (Frankfurt)
    "model_type": "claude",
    "model_id": "anthropic.claude-instant-v1",
    "maxOutputTokens": "8196"
    },
];

const profile_of_LLMs = claude_instant;
```

S3를 정의합니다. bucket 이름은 중복 방지를 위해 주석 처리하였습니다. 고정된 bucket이름을 적용할때에는 주석을 지우고 고유한 이름으로 변경합니다.

```typescript
const bucketName = `storage-for-${projectName}-${region}`;

const s3Bucket = new s3.Bucket(this, `storage-${projectName}`, {
    // bucketName: bucketName,
    blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    removalPolicy: cdk.RemovalPolicy.DESTROY,
    autoDeleteObjects: true,
    publicReadAccess: false,
    versioned: false,
    cors: [
        {
            allowedHeaders: ['*'],
            allowedMethods: [
                s3.HttpMethods.POST,
                s3.HttpMethods.PUT,
            ],
            allowedOrigins: ['*'],
        },
    ],
});
```

Kendra의 FAQ를 위한 셈플을 복사합니다.

```typescript
new s3Deploy.BucketDeployment(this, `upload-contents-for-${projectName}`, {
    sources: [
        s3Deploy.Source.asset("../contents/faq/")
    ],
    destinationBucket: s3Bucket,
    destinationKeyPrefix: 'faq/'
});   
```

CloudFront를 정의합니다.

```typescript
const distribution = new cloudFront.Distribution(this, `cloudfront-for-${projectName}`, {
    defaultBehavior: {
        origin: new origins.S3Origin(s3Bucket),
        allowedMethods: cloudFront.AllowedMethods.ALLOW_ALL,
        cachePolicy: cloudFront.CachePolicy.CACHING_DISABLED,
        viewerProtocolPolicy: cloudFront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    },
    priceClass: cloudFront.PriceClass.PRICE_CLASS_200,
});
new cdk.CfnOutput(this, `distributionDomainName-for-${projectName}`, {
    value: distribution.domainName,
    description: 'The domain name of the Distribution',
});
```

채팅 이력을 저장하기 위해 DynamoDB를 이용합니다.

```typescript
const callLogTableName = `db-call-log-for-${projectName}`;
const callLogDataTable = new dynamodb.Table(this, `db-call-log-for-${projectName}`, {
    tableName: callLogTableName,
    partitionKey: { name: 'user_id', type: dynamodb.AttributeType.STRING },
    sortKey: { name: 'request_time', type: dynamodb.AttributeType.STRING },
    billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
    removalPolicy: cdk.RemovalPolicy.DESTROY,
});
const callLogIndexName = `index-type-for-${projectName}`;
callLogDataTable.addGlobalSecondaryIndex({ // GSI
    indexName: callLogIndexName,
    partitionKey: { name: 'request_id', type: dynamodb.AttributeType.STRING },
});
```

Lambda(Websocket)을 위한 Role을 설정합니다.

```typescript
// Lambda - chat (websocket)
const roleLambdaWebsocket = new iam.Role(this, `role-lambda-chat-ws-for-${projectName}`, {
    roleName: `role-lambda-chat-ws-for-${projectName}-${region}`,
    assumedBy: new iam.CompositePrincipal(
        new iam.ServicePrincipal("lambda.amazonaws.com"),
        new iam.ServicePrincipal("bedrock.amazonaws.com"),
        new iam.ServicePrincipal("kendra.amazonaws.com")
    )
});
roleLambdaWebsocket.addManagedPolicy({
    managedPolicyArn: 'arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole',
});
const BedrockPolicy = new iam.PolicyStatement({  // policy statement for sagemaker
    resources: ['*'],
    actions: ['bedrock:*'],
});
roleLambdaWebsocket.attachInlinePolicy( // add bedrock policy
    new iam.Policy(this, `bedrock-policy-lambda-chat-ws-for-${projectName}`, {
        statements: [BedrockPolicy],
    }),
);
const apiInvokePolicy = new iam.PolicyStatement({
    // resources: ['arn:aws:execute-api:*:*:*'],
    resources: ['*'],
    actions: [
        'execute-api:Invoke',
        'execute-api:ManageConnections'
    ],
});
roleLambdaWebsocket.attachInlinePolicy(
    new iam.Policy(this, `api-invoke-policy-for-${projectName}`, {
        statements: [apiInvokePolicy],
    }),
);
```

Kendra를 정의합니다.

```typescript
let kendraIndex = "";
const roleKendra = new iam.Role(this, `role-kendra-for-${projectName}`, {
    roleName: `role-kendra-for-${projectName}-${region}`,
    assumedBy: new iam.CompositePrincipal(
        new iam.ServicePrincipal("kendra.amazonaws.com")
    )
});
const cfnIndex = new kendra.CfnIndex(this, 'MyCfnIndex', {
    edition: 'DEVELOPER_EDITION',  // ENTERPRISE_EDITION, 
    name: `reg-kendra-${projectName}`,
    roleArn: roleKendra.roleArn,
});
const kendraLogPolicy = new iam.PolicyStatement({
    resources: ['*'],
    actions: ["logs:*", "cloudwatch:GenerateQuery"],
});
roleKendra.attachInlinePolicy( // add kendra policy
    new iam.Policy(this, `kendra-log-policy-for-${projectName}`, {
        statements: [kendraLogPolicy],
    }),
);
const kendraS3ReadPolicy = new iam.PolicyStatement({
    resources: ['*'],
    actions: ["s3:Get*", "s3:List*", "s3:Describe*"],
});
roleKendra.attachInlinePolicy( // add kendra policy
    new iam.Policy(this, `kendra-s3-read-policy-for-${projectName}`, {
        statements: [kendraS3ReadPolicy],
    }),
);

const accountId = process.env.CDK_DEFAULT_ACCOUNT;
const kendraResourceArn = `arn:aws:kendra:${kendra_region}:${accountId}:index/${cfnIndex.attrId}`

const kendraPolicy = new iam.PolicyStatement({
    resources: [kendraResourceArn],
    actions: ['kendra:*'],
});
roleKendra.attachInlinePolicy( // add kendra policy
    new iam.Policy(this, `kendra-inline-policy-for-${projectName}`, {
        statements: [kendraPolicy],
    }),
);
kendraIndex = cfnIndex.attrId;

roleLambdaWebsocket.attachInlinePolicy(
    new iam.Policy(this, `lambda-inline-policy-for-kendra-in-${projectName}`, {
        statements: [kendraPolicy],
    }),
);

const passRoleResourceArn = roleLambdaWebsocket.roleArn;
const passRolePolicy = new iam.PolicyStatement({
    resources: [passRoleResourceArn],
    actions: ['iam:PassRole'],
});

roleLambdaWebsocket.attachInlinePolicy( // add pass role policy
    new iam.Policy(this, `pass-role-of-kendra-for-${projectName}`, {
        statements: [passRolePolicy],
    }),
); 
```

Polly를 위한 권한을 Lambda에 추가합니다.

```typescript
// Poly Role
const PollyPolicy = new iam.PolicyStatement({
    actions: ['polly:*'],
    resources: ['*'],
});
roleLambdaWebsocket.attachInlinePolicy(
    new iam.Policy(this, 'polly-policy', {
        statements: [PollyPolicy],
    }),
);
```

Kendra에서 AWS CLI로 data source를 추가하기 위한 명령어를 아래와 같이 Output으로 출력합니다.

```typescript
new cdk.CfnOutput(this, `create-S3-data-source-for-${projectName}`, {
    value: 'aws kendra create-data-source --index-id ' + kendraIndex + ' --name data-source-for-upload-file --type S3 --role-arn ' + roleLambdaWebsocket.roleArn + ' --configuration \'{\"S3Configuration\":{\"BucketName\":\"' + s3Bucket.bucketName + '\", \"DocumentsMetadataConfiguration\": {\"S3Prefix\":\"metadata/\"},\"InclusionPrefixes\": [\"' + s3_prefix + '\"]}}\' --language-code ko --region ' + kendra_region,
    description: 'The commend to create data source using S3',
});
```

OpenSearch를 정의합니다.

```typescript
// opensearch
// Permission for OpenSearch
const domainName = projectName
const resourceArn = `arn:aws:es:${region}:${accountId}:domain/${domainName}/*`
if (debug) {
    new cdk.CfnOutput(this, `resource-arn-for-${projectName}`, {
        value: resourceArn,
        description: 'The arn of resource',
    });
}

const OpenSearchAccessPolicy = new iam.PolicyStatement({
    resources: [resourceArn],
    actions: ['es:*'],
    effect: iam.Effect.ALLOW,
    principals: [new iam.AnyPrincipal()],
});

const domain = new opensearch.Domain(this, 'Domain', {
    version: opensearch.EngineVersion.OPENSEARCH_2_3,

    domainName: domainName,
    removalPolicy: cdk.RemovalPolicy.DESTROY,
    enforceHttps: true,
    fineGrainedAccessControl: {
        masterUserName: opensearch_account,
        // masterUserPassword: cdk.SecretValue.secretsManager('opensearch-private-key'),
        masterUserPassword: cdk.SecretValue.unsafePlainText(opensearch_passwd)
    },
    capacity: {
        masterNodes: 3,
        masterNodeInstanceType: 'r6g.large.search',
        // multiAzWithStandbyEnabled: false,
        dataNodes: 15,
        dataNodeInstanceType: 'r6g.large.search',
        // warmNodes: 2,
        // warmInstanceType: 'ultrawarm1.medium.search',
    },
    accessPolicies: [OpenSearchAccessPolicy],
    ebs: {
        volumeSize: 100,
        volumeType: ec2.EbsDeviceVolumeType.GP3,
    },
    nodeToNodeEncryption: true,
    encryptionAtRest: {
        enabled: true,
    },
    zoneAwareness: {
        enabled: true,
        availabilityZoneCount: 3,
    }
});

opensearch_url = 'https://'+domain.domainEndpoint;
```

Restful API를 위한 API Gateway를 정의합니다.

```typescript
// api role
const role = new iam.Role(this, `api-role-for-${projectName}`, {
    roleName: `api-role-for-${projectName}-${region}`,
    assumedBy: new iam.ServicePrincipal("apigateway.amazonaws.com")
});
role.addToPolicy(new iam.PolicyStatement({
    resources: ['*'],
    actions: [
        'lambda:InvokeFunction',
        'cloudwatch:*'
    ]
}));
role.addManagedPolicy({
    managedPolicyArn: 'arn:aws:iam::aws:policy/AWSLambdaExecute',
});

// API Gateway
const api = new apiGateway.RestApi(this, `api-chatbot-for-${projectName}`, {
    description: 'API Gateway for chatbot',
    endpointTypes: [apiGateway.EndpointType.REGIONAL],
    binaryMediaTypes: ['application/pdf', 'text/plain', 'text/csv', 'application/vnd.ms-powerpoint', 'application/vnd.ms-excel', 'application/msword'],
    deployOptions: {
        stageName: stage,

        // logging for debug
        // loggingLevel: apiGateway.MethodLoggingLevel.INFO, 
        // dataTraceEnabled: true,
    },
});
```

Lambda (upload)를 정의합니다. 이것은 presigned url을 생성하여 client에서 큰 파일을 S3로 직접 업로드 할 수 있도록 해줍니다. 

```typescript
// Lambda - Upload
const lambdaUpload = new lambda.Function(this, `lambda-upload-for-${projectName}`, {
    runtime: lambda.Runtime.NODEJS_16_X,
    functionName: `lambda-upload-for-${projectName}`,
    code: lambda.Code.fromAsset("../lambda-upload"),
    handler: "index.handler",
    timeout: cdk.Duration.seconds(10),
    logRetention: logs.RetentionDays.ONE_DAY,
    environment: {
        bucketName: s3Bucket.bucketName,
        s3_prefix: s3_prefix
    }
});
s3Bucket.grantReadWrite(lambdaUpload);

// POST method - upload
const resourceName = "upload";
const upload = api.root.addResource(resourceName);
upload.addMethod('POST', new apiGateway.LambdaIntegration(lambdaUpload, {
    passthroughBehavior: apiGateway.PassthroughBehavior.WHEN_NO_TEMPLATES,
    credentialsRole: role,
    integrationResponses: [{
        statusCode: '200',
    }],
    proxy: false,
}), {
    methodResponses: [
        {
            statusCode: '200',
            responseModels: {
                'application/json': apiGateway.Model.EMPTY_MODEL,
            },
        }
    ]
});
if (debug) {
    new cdk.CfnOutput(this, `ApiGatewayUrl-for-${projectName}`, {
        value: api.url + 'upload',
        description: 'The url of API Gateway',
    });
}

// cloudfront setting  
distribution.addBehavior("/upload", new origins.RestApiOrigin(api), {
    cachePolicy: cloudFront.CachePolicy.CACHING_DISABLED,
    allowedMethods: cloudFront.AllowedMethods.ALLOW_ALL,
    viewerProtocolPolicy: cloudFront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
});    
```

Lambda (query)를 정의합니다. 이것은 client에서 DynamoDB에 저장된 결과를 가져오는데 활용됩니다.

```typescript
// Lambda - queryResult
    const lambdaQueryResult = new lambda.Function(this, `lambda-query-for-${projectName}`, {
      runtime: lambda.Runtime.NODEJS_16_X, 
      functionName: `lambda-query-for-${projectName}`,
      code: lambda.Code.fromAsset("../lambda-query"), 
      handler: "index.handler", 
      timeout: cdk.Duration.seconds(60),
      logRetention: logs.RetentionDays.ONE_DAY,
      environment: {
        tableName: callLogTableName,
        indexName: callLogIndexName
      }      
    });
    callLogDataTable.grantReadWriteData(lambdaQueryResult); // permission for dynamo
    
    // POST method - query
    const query = api.root.addResource("query");
    query.addMethod('POST', new apiGateway.LambdaIntegration(lambdaQueryResult, {
      passthroughBehavior: apiGateway.PassthroughBehavior.WHEN_NO_TEMPLATES,
      credentialsRole: role,
      integrationResponses: [{
        statusCode: '200',
      }], 
      proxy:false, 
    }), {
      methodResponses: [  
        {
          statusCode: '200',
          responseModels: {
            'application/json': apiGateway.Model.EMPTY_MODEL,
          }, 
        }
      ]
    }); 

    // cloudfront setting for api gateway    
    distribution.addBehavior("/query", new origins.RestApiOrigin(api), {
      cachePolicy: cloudFront.CachePolicy.CACHING_DISABLED,
      allowedMethods: cloudFront.AllowedMethods.ALLOW_ALL,  
      viewerProtocolPolicy: cloudFront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    });
```

Lambda (history)는 Client에서 채팅 이력을 가져올때 이용합니다. 

```typescript
// Lambda - getHistory
const lambdaGetHistory = new lambda.Function(this, `lambda-gethistory-for-${projectName}`, {
    runtime: lambda.Runtime.NODEJS_16_X,
    functionName: `lambda-gethistory-for-${projectName}`,
    code: lambda.Code.fromAsset("../lambda-gethistory"),
    handler: "index.handler",
    timeout: cdk.Duration.seconds(60),
    logRetention: logs.RetentionDays.ONE_DAY,
    environment: {
        tableName: callLogTableName
    }
});
callLogDataTable.grantReadWriteData(lambdaGetHistory); // permission for dynamo

// POST method - history
const history = api.root.addResource("history");
history.addMethod('POST', new apiGateway.LambdaIntegration(lambdaGetHistory, {
    passthroughBehavior: apiGateway.PassthroughBehavior.WHEN_NO_TEMPLATES,
    credentialsRole: role,
    integrationResponses: [{
        statusCode: '200',
    }],
    proxy: false,
}), {
    methodResponses: [
        {
            statusCode: '200',
            responseModels: {
                'application/json': apiGateway.Model.EMPTY_MODEL,
            },
        }
    ]
});

// cloudfront setting for api gateway    
distribution.addBehavior("/history", new origins.RestApiOrigin(api), {
    cachePolicy: cloudFront.CachePolicy.CACHING_DISABLED,
    allowedMethods: cloudFront.AllowedMethods.ALLOW_ALL,
    viewerProtocolPolicy: cloudFront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
});
```

채팅이력을 삭제할때 사용한 Lambda (delete)를 정의합니다. 

```typescript
// Lambda - deleteItems
const lambdaDeleteItems = new lambda.Function(this, `lambda-deleteItems-for-${projectName}`, {
    runtime: lambda.Runtime.NODEJS_16_X,
    functionName: `lambda-deleteItems-for-${projectName}`,
    code: lambda.Code.fromAsset("../lambda-delete-items"),
    handler: "index.handler",
    timeout: cdk.Duration.seconds(60),
    logRetention: logs.RetentionDays.ONE_DAY,
    environment: {
        tableName: callLogTableName
    }
});
callLogDataTable.grantReadWriteData(lambdaDeleteItems); // permission for dynamo

// POST method - delete items
const deleteItem = api.root.addResource("delete");
deleteItem.addMethod('POST', new apiGateway.LambdaIntegration(lambdaDeleteItems, {
    passthroughBehavior: apiGateway.PassthroughBehavior.WHEN_NO_TEMPLATES,
    credentialsRole: role,
    integrationResponses: [{
        statusCode: '200',
    }],
    proxy: false,
}), {
    methodResponses: [
        {
            statusCode: '200',
            responseModels: {
                'application/json': apiGateway.Model.EMPTY_MODEL,
            },
        }
    ]
});

// cloudfront setting for api gateway    
distribution.addBehavior("/delete", new origins.RestApiOrigin(api), {
    cachePolicy: cloudFront.CachePolicy.CACHING_DISABLED,
    allowedMethods: cloudFront.AllowedMethods.ALLOW_ALL,
    viewerProtocolPolicy: cloudFront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
});
```

Websocket을 이용할 수 있도록 API Gateway를 정의합니다. 

```typescript
/ stream api gateway
// API Gateway
const websocketapi = new apigatewayv2.CfnApi(this, `ws-api-for-${projectName}`, {
    description: 'API Gateway for chatbot using websocket',
    apiKeySelectionExpression: "$request.header.x-api-key",
    name: 'api-' + projectName,
    protocolType: "WEBSOCKET", // WEBSOCKET or HTTP
    routeSelectionExpression: "$request.body.action",
});
websocketapi.applyRemovalPolicy(cdk.RemovalPolicy.DESTROY); // DESTROY, RETAIN

const wss_url = `wss://${websocketapi.attrApiId}.execute-api.${region}.amazonaws.com/${stage}`;
new cdk.CfnOutput(this, 'web-socket-url', {
    value: wss_url,
    description: 'The URL of Web Socket',
});

const connection_url = `https://${websocketapi.attrApiId}.execute-api.${region}.amazonaws.com/${stage}`;
if (debug) {
    new cdk.CfnOutput(this, 'api-identifier', {
        value: websocketapi.attrApiId,
        description: 'The API identifier.',
    });

    new cdk.CfnOutput(this, 'connection-url', {
        value: connection_url,
        description: 'The URL of connection',
    });
}
```

구글 API를 사용할때 사용하는 Secret를 저장하기 위해 Seceret Manager를 설정합니다. 

```typescript
const googleApiSecret = new secretsmanager.Secret(this, `google-api-secret-for-${projectName}`, {
    description: 'secret for google api key',
    removalPolicy: cdk.RemovalPolicy.DESTROY,
    secretName: 'googl_api_key',
    generateSecretString: {
        secretStringTemplate: JSON.stringify({
            google_cse_id: 'cse_id'
        }),
        generateStringKey: 'google_api_key',
        excludeCharacters: '/@"',
    },

});
googleApiSecret.grantRead(roleLambdaWebsocket) 
```

Lambda(Websocket)을 정의합니다. 

```typescript
// lambda-chat using websocket    
const lambdaChatWebsocket = new lambda.DockerImageFunction(this, `lambda-chat-ws-for-${projectName}`, {
    description: 'lambda for chat using websocket',
    functionName: `lambda-chat-ws-for-${projectName}`,
    code: lambda.DockerImageCode.fromImageAsset(path.join(__dirname, '../../lambda-chat-ws')),
    timeout: cdk.Duration.seconds(300),
    memorySize: 8192,
    role: roleLambdaWebsocket,
    environment: {
        // bedrock_region: bedrock_region,
        kendra_region: String(kendra_region),
        // model_id: model_id,
        s3_bucket: s3Bucket.bucketName,
        s3_prefix: s3_prefix,
        callLogTableName: callLogTableName,
        connection_url: connection_url,
        enableReference: enableReference,
        opensearch_account: opensearch_account,
        opensearch_passwd: opensearch_passwd,
        opensearch_url: opensearch_url,
        path: 'https://' + distribution.domainName + '/',
        kendraIndex: kendraIndex,
        roleArn: roleLambdaWebsocket.roleArn,
        debugMessageMode: debugMessageMode,
        rag_method: rag_method,
        useParallelRAG: useParallelRAG,
        numberOfRelevantDocs: numberOfRelevantDocs,
        kendraMethod: kendraMethod,
        profile_of_LLMs: JSON.stringify(profile_of_LLMs),
        capabilities: capabilities,
        googleApiSecret: googleApiSecret.secretName,
        allowDualSearch: allowDualSearch,
        enableNoriPlugin: enableNoriPlugin
    }
});
lambdaChatWebsocket.grantInvoke(new iam.ServicePrincipal('apigateway.amazonaws.com'));
s3Bucket.grantReadWrite(lambdaChatWebsocket); // permission for s3
callLogDataTable.grantReadWriteData(lambdaChatWebsocket); // permission for dynamo 
```

Websocket을 처리하기 위한 API gateway를 설정합니다.

```typescript
const integrationUri = `arn:aws:apigateway:${region}:lambda:path/2015-03-31/functions/${lambdaChatWebsocket.functionArn}/invocations`;
const cfnIntegration = new apigatewayv2.CfnIntegration(this, `api-integration-for-${projectName}`, {
    apiId: websocketapi.attrApiId,
    integrationType: 'AWS_PROXY',
    credentialsArn: role.roleArn,
    connectionType: 'INTERNET',
    description: 'Integration for connect',
    integrationUri: integrationUri,
});

new apigatewayv2.CfnRoute(this, `api-route-for-${projectName}-connect`, {
    apiId: websocketapi.attrApiId,
    routeKey: "$connect",
    apiKeyRequired: false,
    authorizationType: "NONE",
    operationName: 'connect',
    target: `integrations/${cfnIntegration.ref}`,
});

new apigatewayv2.CfnRoute(this, `api-route-for-${projectName}-disconnect`, {
    apiId: websocketapi.attrApiId,
    routeKey: "$disconnect",
    apiKeyRequired: false,
    authorizationType: "NONE",
    operationName: 'disconnect',
    target: `integrations/${cfnIntegration.ref}`,
});

new apigatewayv2.CfnRoute(this, `api-route-for-${projectName}-default`, {
    apiId: websocketapi.attrApiId,
    routeKey: "$default",
    apiKeyRequired: false,
    authorizationType: "NONE",
    operationName: 'default',
    target: `integrations/${cfnIntegration.ref}`,
});

new apigatewayv2.CfnStage(this, `api-stage-for-${projectName}`, {
    apiId: websocketapi.attrApiId,
    stageName: stage
});

new cdk.CfnOutput(this, `FAQ-Update-for-${projectName}`, {
    value: 'aws kendra create-faq --index-id ' + kendraIndex + ' --name faq-banking --s3-path \'{\"Bucket\":\"' + s3Bucket.bucketName + '\", \"Key\":\"faq/faq-banking.csv\"}\' --role-arn ' + roleLambdaWebsocket.roleArn + ' --language-code ko --region ' + kendra_region + ' --file-format CSV',
    description: 'The commend for uploading contents of FAQ',
});
```

다수의 문서들이 S3로 인입될때 손실없이 처리하기 위해 SQS를 이용합니다. 

```typescript
// SQS for S3 event (fifo) 
let queueUrl: string[] = [];
let queue: any[] = [];
for (let i = 0; i < profile_of_LLMs.length; i++) {
    queue[i] = new sqs.Queue(this, 'QueueS3EventFifo' + i, {
        visibilityTimeout: cdk.Duration.seconds(600),
        queueName: `queue-s3-event-for-${projectName}-${i}.fifo`,
        fifo: true,
        contentBasedDeduplication: false,
        deliveryDelay: cdk.Duration.millis(0),
        retentionPeriod: cdk.Duration.days(2),
    });
    queueUrl.push(queue[i].queueUrl);
}

// Lambda for s3 event manager
const lambdaS3eventManager = new lambda.Function(this, `lambda-s3-event-manager-for-${projectName}`, {
    description: 'lambda for s3 event manager',
    functionName: `lambda-s3-event-manager-for-${projectName}`,
    handler: 'lambda_function.lambda_handler',
    runtime: lambda.Runtime.PYTHON_3_11,
    code: lambda.Code.fromAsset(path.join(__dirname, '../../lambda-s3-event-manager')),
    timeout: cdk.Duration.seconds(60),
    logRetention: logs.RetentionDays.ONE_DAY,
    environment: {
        sqsFifoUrl: JSON.stringify(queueUrl),
        nqueue: String(profile_of_LLMs.length)
    }
});
for (let i = 0; i < profile_of_LLMs.length; i++) {
    queue[i].grantSendMessages(lambdaS3eventManager); // permision for SQS putItem
}

// Lambda for document manager
let lambdDocumentManager: any[] = [];
for (let i = 0; i < profile_of_LLMs.length; i++) {
    lambdDocumentManager[i] = new lambda.DockerImageFunction(this, `lambda-document-manager-for-${projectName}-${i}`, {
        description: 'S3 document manager',
        functionName: `lambda-document-manager-for-${projectName}-${i}`,
        role: roleLambdaWebsocket,
        code: lambda.DockerImageCode.fromImageAsset(path.join(__dirname, '../../lambda-document-manager')),
        timeout: cdk.Duration.seconds(600),
        memorySize: 8192,
        environment: {
            bedrock_region: profile_of_LLMs[i].bedrock_region,
            s3_bucket: s3Bucket.bucketName,
            s3_prefix: s3_prefix,
            kendra_region: String(kendra_region),
            opensearch_account: opensearch_account,
            opensearch_passwd: opensearch_passwd,
            opensearch_url: opensearch_url,
            kendraIndex: kendraIndex,
            roleArn: roleLambdaWebsocket.roleArn,
            path: 'https://' + distribution.domainName + '/',
            capabilities: capabilities,
            sqsUrl: queueUrl[i],
            max_object_size: String(max_object_size),
            enableNoriPlugin: enableNoriPlugin,
            supportedFormat: supportedFormat,
            profile_of_LLMs: JSON.stringify(profile_of_LLMs),
            enableParallelSummay: enableParallelSummay
        }
    });
    s3Bucket.grantReadWrite(lambdDocumentManager[i]); // permission for s3
    lambdDocumentManager[i].addEventSource(new SqsEventSource(queue[i])); // permission for SQS
}

// s3 event source
const s3PutEventSource = new lambdaEventSources.S3EventSource(s3Bucket, {
    events: [
        s3.EventType.OBJECT_CREATED_PUT,
        s3.EventType.OBJECT_REMOVED_DELETE
    ],
    filters: [
        { prefix: s3_prefix + '/' },
    ]
});
lambdaS3eventManager.addEventSource(s3PutEventSource); 
```

Lambda (Provisioning)은 Websocket의 endpoint 정보를 client에게 전달하기 위해 만든 Lambda입니다. 

```typescript
// lambda - provisioning
const lambdaProvisioning = new lambda.Function(this, `lambda-provisioning-for-${projectName}`, {
    description: 'lambda to earn provisioning info',
    functionName: `lambda-provisioning-api-${projectName}`,
    handler: 'lambda_function.lambda_handler',
    runtime: lambda.Runtime.PYTHON_3_11,
    code: lambda.Code.fromAsset(path.join(__dirname, '../../lambda-provisioning')),
    timeout: cdk.Duration.seconds(30),
    logRetention: logs.RetentionDays.ONE_DAY,
    environment: {
        wss_url: wss_url,
    }
});

// POST method - provisioning
const provisioning_info = api.root.addResource("provisioning");
provisioning_info.addMethod('POST', new apiGateway.LambdaIntegration(lambdaProvisioning, {
    passthroughBehavior: apiGateway.PassthroughBehavior.WHEN_NO_TEMPLATES,
    credentialsRole: role,
    integrationResponses: [{
        statusCode: '200',
    }],
    proxy: false,
}), {
    methodResponses: [
        {
            statusCode: '200',
            responseModels: {
                'application/json': apiGateway.Model.EMPTY_MODEL,
            },
        }
    ]
});

// cloudfront setting for provisioning api
distribution.addBehavior("/provisioning", new origins.RestApiOrigin(api), {
    cachePolicy: cloudFront.CachePolicy.CACHING_DISABLED,
    allowedMethods: cloudFront.AllowedMethods.ALLOW_ALL,
    viewerProtocolPolicy: cloudFront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
});
```

API Gateway를 배포합니다.

```typescript
// deploy components
new componentDeployment(scope, `component-deployment-of-${projectName}`, websocketapi.attrApiId)

export class componentDeployment extends cdk.Stack {
    constructor(scope: Construct, id: string, appId: string, props?: cdk.StackProps) {
        super(scope, id, props);

        new apigatewayv2.CfnDeployment(this, `api-deployment-of-${projectName}`, {
            apiId: appId,
            description: "deploy api gateway using websocker",  // $default
            stageName: stage
        });
    }
} 
```
