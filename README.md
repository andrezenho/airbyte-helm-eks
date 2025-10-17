a politica do service account tem essas permissoes
no trust relantionship:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<MY-ACCOUNT>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<OIDC-CLUSTER-NUMBER>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-1.amazonaws.com/id/<OIDC-CLUSTER-NUMBER>:sub": "system:serviceaccount:airbyte:airbyte-sa",
                    "oidc.eks.us-east-1.amazonaws.com/id/<OIDC-CLUSTER-NUMBER>:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}

# ABAIXO A POLITICA PARA ACESSO NA BUCKET:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ]
        }
    ]
}