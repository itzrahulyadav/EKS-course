## Create cluster using Cloudformation template.

1. Go to s3 console and create a new s3 bucket.
2. Upload the cloudformation.yaml file downloaded from the github
3. Go to IAM page on AWS Console
4. Create an IAM role with cloudformation as the trusted principal.
5. Add AdministratorAccess permissions policy.
6. Copy the objects url which was uploaded to s3
7. Go to Cloudformation console 
8. Select create stack
9. Select choose an existing template
10. Select Amazon S3 URL in the specify template
11. Paste the object url 
12. Provide name to the stack
13. Leave the parameters as they are
14. Select the IAM role created earlier in the permissions tab
15. Select `Preserve successfully provisioned resources` in the Stack failure options
16. Select the checkbox `I acknowledge that AWS CloudFormation might create IAM resources with custom names` to allow cloudformation to create resources in your behalf.
17. Scroll to the bottom in the Review and create page and click on create.
