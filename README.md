# MovieLens Sample

The MovieLens sample demonstrates how to build personalized recommendation
models to recommend movies to users based on movie ratings data from [MovieLens 20M dataset](https://grouplens.org/datasets/movielens/20m/).

## PREPROCESS

PROJECT=frc-cloudml

BUCKET=gs://frc-cloudml-ml

GCS_TRAINING_INPUT_DIR=gs://frc-cloudml-ml-20m  # the csv files are here

PREPROCESS_OUTPUT="${BUCKET}/preprocess"

`python preprocess.py --input_dir "${GCS_TRAINING_INPUT_DIR}" \
                     --output_dir "${PREPROCESS_OUTPUT}" \
                     --percent_eval 20 \
                     --project_id frc-cloudml \
                     --negative_sample_ratio 1 \
                     --negative_sample_label 0.0 \
                     --eval_type ranking \
                     --eval_score_threshold 4.5 \
                     --num_ranking_candidate_movie_ids 1000 \
                     --partition_random_seed 0 \
                     --cloud`

^ This runs for about 30 minutes on ~200 compute nodes
(GCP Dataflow, aka. Apache Beam pipeline)

####
## TRAINING

JOB_ID=xxx

TRAINING_OUTPUT="${BUCKET}/model/${JOB_ID}"

`gcloud ml-engine jobs submit training "${JOB_ID}" \
                      --stream-logs \
                      --module-name trainer.task \
                      --package-path trainer \
                      --staging-bucket "${BUCKET}" \
                      --region us-central1 \
                      --config config.yaml \
                      -- \
                      --raw_metadata_path "${PREPROCESS_OUTPUT}/raw_metadata" \
                      --transform_savedmodel "${PREPROCESS_OUTPUT}/transform_fn" \
                      --eval_data_paths "${PREPROCESS_OUTPUT}/features_eval*.tfrecord.gz" \
                      --train_data_paths "${PREPROCESS_OUTPUT}/features_train*.tfrecord.gz" \
                      --output_path "${TRAINING_OUTPUT}" \
                      --model_type dnn_softmax \
                      --eval_type ranking \
                      --l2_weight_decay 0.01 \
                      --learning_rate 0.05 \
                      --train_steps 500000 \
                      --eval_steps 500 \
                      --top_k_infer 100`

^ This runs for 10 hours and consumes about 80 ML units with the
default settings (standard_gpu, 2 workers)

## DEPLOY

an unique folder (eg. "1545291911") is created here by the training process
it contains the trained model (it's just a file in a bucket - you could also
copy it to local workstation and run locally, or even in a browser.)

`gsutil ls "${TRAINING_OUTPUT}/export/Servo"`

`gcloud ml-engine models create "movielens" --regions us-central1`

`gcloud ml-engine versions create "v1" --model "movielens" --origin "${TRAINING_OUTPUT}/export/Servo/1545291911"`

## PREDICTION (CLI)

The preprocess script generates sets of tf.Example records.
These can be used for testing the prediction, but ultimately you'd want to
build your own (eg. based on user input, in real-time). Format is base64 encoded
tfrecords in a json wrapper eg. {"b64": "CpCGAQ..."}

The input instance has values eg. the movie ids and ratings (by this new, formely
unknown user). The result is list of movie ids and their _predicted_ ratings

LOCAL_PREDICTION_FILE="feat1.json"

`gcloud ml-engine predict --model "movielens" --version "v1" --json-instances "${LOCAL_PREDICTION_FILE}"`

output:
`CLASSES                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         SCORES
[u'495', u'1124', u'1034', u'1844', u'2118', u'1142', u'947', u'672', u'33', u'1072', u'160', u'872', u'803', u'583', u'911', u'478', u'747', u'928', u'2361', u'299', u'1162', u'1996', u'255', u'1930', u'49', u'1857', u'1241', u'1518', u'197', u'416', u'363', u'592', u'554', u'229', u'2006', u'1277', u'871', u'1046', u'68', u'1427', u'832', u'269', u'1338', u'1849', u'2864', u'938', u'794', u'1653', u'1678', u'522', u'866', u'1210', u'1173', u'1599', u'2816', u'705', u'1043', u'1957', u'1257', u'894', u'2182', u'1529', u'791', u'2501', u'1604', u'446', u'1470', u'31', u'462', u'728', u'1294', u'2402', u'438', u'948', u'736', u'383', u'1375', u'2283', u'2112', u'823', u'1645', u'869', u'812', u'1199', u'801', u'760', u'1191', u'1721', u'2685', u'3433', u'1628', u'1785', u'84', u'602', u'463', u'1502', u'322', u'1921', u'1550', u'2708']  [0.004149864427745342, 0.002873373217880726, 0.0025119830388575792, 0.001956462161615491, 0.0018452743533998728, 0.001791705610230565, 0.0017159412382170558, 0.0016688746400177479, 0.0015741839306429029, 0.0014790329150855541, 0.0014661855529993773, 0.0014058025553822517, 0.001396586885675788, 0.0013659590622410178, 0.0013335536932572722, 0.001223740284331143, 0.0011684602359309793, 0.0011337596224620938, 0.0011198982829228044, 0.0011142333969473839, 0.001078687608242035, 0.0010275953682139516, 0.0010010827099904418, 0.000999882468022406, 0.000984544982202351, 0.0009664074750617146, 0.0009551883558742702, 0.0009529407834634185, 0.0009440299472771585, 0.000918190402444452, 0.0009097539004869759, 0.0009019459830597043, 0.0009012406808324158, 0.0008962687570601702, 0.0008684145286679268, 0.0008588095079176128, 0.0008506296435371041, 0.0008280483307316899, 0.000821010151412338, 0.0008141801808960736, 0.0008057533414103091, 0.0007947291014716029, 0.0007913510780781507, 0.0007885681698098779, 0.0007876880117692053, 0.0007788035436533391, 0.0007686662138439715, 0.0007632080232724547, 0.0007625276339240372, 0.0007532995077781379, 0.0007432583370245993, 0.0007429036195389926, 0.0007417617016471922, 0.0007243551081046462, 0.0007231704657897353, 0.0007209368632175028, 0.0007203167187981308, 0.0007151925819925964, 0.0007024531369097531, 0.0007016970776021481, 0.0006817614194005728, 0.0006705757696181536, 0.0006695378106087446, 0.0006692272145301104, 0.0006642197258770466, 0.0006576593732461333, 0.0006572592537850142, 0.0006561484187841415, 0.0006455555558204651, 0.0006427497137337923, 0.0006402853759936988, 0.0006399346166290343, 0.0006387559114955366, 0.0006362585118040442, 0.000627643836196512, 0.00062736333347857, 0.0006271590245887637, 0.0006270006997510791, 0.0006074020639061928, 0.0006035408587194979, 0.0006030527292750776, 0.0005842868122272193, 0.0005825633998028934, 0.0005710627301596105, 0.0005704254726879299, 0.0005702057387679815, 0.0005692029953934252, 0.0005652288673445582, 0.0005623499746434391, 0.0005622367025353014, 0.0005599004216492176, 0.000557517574634403, 0.0005539593985304236, 0.0005501905689015985, 0.0005468948511406779, 0.0005463325069285929, 0.0005462242988869548, 0.0005416521453298628, 0.0005402805982157588, 0.0005387062556110322]`

^ This prediction runs in ~800 ms, in this case

See prediction.js (below) for example how to make predictions thru the
REST API. Factoring this to a cloud function (https trigger) makes
it completely serverless and easy to access from any eg. browser-based
Angular app.

```import * as functions from 'firebase-functions';

import { google } from 'googleapis';
const ml = google.ml('v1');

export const predictMovie = functions.https.onRequest(async (request, response) => {
  const instances = request.body.instances;
  const model = request.body.model;

  const { credential } = await google.auth.getApplicationDefault();
  const modelName = `projects/YOUR-PROJECT/models/${model}`;

  const preds = await ml.projects.predict({
    auth: credential,
    name: modelName,
    requestBody: { instances }
  } as any);

  response.send(JSON.stringify(preds.data))
});
