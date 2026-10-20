










csv_reader_unten.py

"""
UNTEN用CSV読込モジュール

仕様
------
・Shift-JIS固定
・アナログとデジタルでレイアウトが異なる
・ヘッダ部(先頭6行)のみ使用
・7行目以降の実績データは使用しない
"""

import csv


def normalize_value(value):
    """ExcelがCSV出力した計測点番号の「="0000000"」を0000000に変換する"""

    value = value.strip()

    if value.startswith('="') and value.endswith('"'):
        value = value[2:-1]

    return value


def read_header_csv(
    file_path,
    file_kind
):
    """CSVのヘッダ情報を読み込む"""

    with open(
        file_path,
        mode="r",
        encoding="shift_jis",
        newline=""
    ) as csv_file:

        rows = list(csv.reader(csv_file))

    # ヘッダ6行のみ利用
    header_rows = rows[:6]

    result = []

    # B列以降を処理
    column_count = len(header_rows[0])

    for col in range(1, column_count):

        # アナログ
        if file_kind == "A":

            record = {
                "machine_name": header_rows[0][col].strip(),
                "message_id": header_rows[1][col].strip(),
                "address": header_rows[2][col].strip(),
                "bit": "",
                "measuring_point_no": normalize_value(header_rows[3][col]),
                "measuring_point_name": header_rows[4][col].strip(),
                "unit": header_rows[5][col].strip(),
            }

        # デジタル
        else:

            record = {
                "machine_name": header_rows[0][col].strip(),
                "message_id": header_rows[1][col].strip(),
                "address": header_rows[2][col].strip(),
                "bit": header_rows[3][col].strip(),
                "measuring_point_no": normalize_value(header_rows[4][col]),
                "measuring_point_name": header_rows[5][col].strip(),
                "unit": "",
            }

        result.append(record)

    return result









importer_unten.py

"""UNTEN取込処理"""

from datetime import datetime
from pathlib import Path

from sqlalchemy import text

from .csv_reader_unten import read_header_csv
from .repository import (
    get_compare_keys,
    get_target_folders,
    get_target_file_patterns
)
from .logger import logger


def build_key(
    record,
    compare_keys
):
    """比較キー生成"""

    values = []

    for key_name in compare_keys:

        value = record.get(key_name)

        if value is None:
            value = ""

        values.append(str(value))

    return "|".join(values)


def resolve_object_names(
    folder_name,
    file_kind
):
    """テーブル名・ビュー名生成"""

    suffix = f"{folder_name.lower()}_{file_kind.lower()}"

    return (
        f"t_headers_{suffix}",
        f"t_versions_{suffix}",
        f"v_latest_header_{suffix}"
    )


def get_latest_version(
    conn,
    version_table,
    ship_no
):
    """最新Version取得"""

    sql = text(
        f"""
        SELECT
            MAX(header_version)
        FROM {version_table}
        WHERE ship_no = :ship_no
        """
    )

    return conn.execute(
        sql,
        {"ship_no": ship_no}
    ).scalar()


def load_latest_headers(
    conn,
    latest_view,
    ship_no,
    compare_keys
):
    """最新ビュー取得"""

    sql = text(
        f"""
        SELECT *
        FROM {latest_view}
        WHERE ship_no = :ship_no
        """
    )

    rows = conn.execute(
        sql,
        {"ship_no": ship_no}
    ).mappings().all()

    result = {}

    for row in rows:

        row_dict = dict(row)

        result[
            build_key(
                row_dict,
                compare_keys
            )
        ] = row_dict

    return result


def extract_datetime_from_filename(
    file_name
):
    """ファイル名末尾日時取得"""

    base_name = Path(file_name).stem

    datetime_str = base_name.split("-")[-1]

    return datetime.strptime(
        datetime_str,
        "%Y%m%d%H%M%S"
    )


def insert_version(
    conn,
    table_name,
    ship_no,
    version,
    created_at
):
    """Version登録"""

    sql = text(
        f"""
        INSERT INTO {table_name}
        (
            ship_no,
            header_version,
            created_at
        )
        VALUES
        (
            :ship_no,
            :header_version,
            :created_at
        )
        """
    )

    conn.execute(
        sql,
        {
            "ship_no": ship_no,
            "header_version": version,
            "created_at": created_at
        }
    )


def insert_header(
    conn,
    table_name,
    record,
    ship_no,
    version,
    seq_no,
    file_type,
    file_kind
):
    """ヘッダ登録"""

    if file_kind == "A":

        sql = text(
            f"""
            INSERT INTO {table_name}
            (
                ship_no,
                header_version,
                seq_no,
                tag_no,
                machine_name,
                message_id,
                address,
                measuring_point_no,
                measuring_point_name,
                unit,
                number_of_decimals,
                convert_tag_no,
                file_type,
                comment
            )
            VALUES
            (
                :ship_no,
                :header_version,
                :seq_no,
                NULL,
                :machine_name,
                :message_id,
                :address,
                :measuring_point_no,
                :measuring_point_name,
                :unit,
                NULL,
                NULL,
                :file_type,
                NULL
            )
            """
        )

        params = {
            "ship_no": ship_no,
            "header_version": version,
            "seq_no": seq_no,
            "machine_name": record["machine_name"],
            "message_id": record["message_id"],
            "address": record["address"],
            "measuring_point_no": record["measuring_point_no"],
            "measuring_point_name": record["measuring_point_name"],
            "unit": record["unit"],
            "file_type": file_type
        }

    else:

        sql = text(
            f"""
            INSERT INTO {table_name}
            (
                ship_no,
                header_version,
                seq_no,
                tag_no,
                machine_name,
                message_id,
                address,
                bit,
                measuring_point_no,
                measuring_point_name,
                number_of_decimals,
                convert_tag_no,
                file_type,
                comment
            )
            VALUES
            (
                :ship_no,
                :header_version,
                :seq_no,
                NULL,
                :machine_name,
                :message_id,
                :address,
                :bit,
                :measuring_point_no,
                :measuring_point_name,
                NULL,
                NULL,
                :file_type,
                NULL
            )
            """
        )

        params = {
            "ship_no": ship_no,
            "header_version": version,
            "seq_no": seq_no,
            "machine_name": record["machine_name"],
            "message_id": record["message_id"],
            "address": record["address"],
            "bit": record["bit"],
            "measuring_point_no": record["measuring_point_no"],
            "measuring_point_name": record["measuring_point_name"],
            "file_type": file_type
        }

    conn.execute(
        sql,
        params
    )


def process_csv_file(
    conn,
    ship_no,
    folder_name,
    csv_file
):
    """CSVファイル単位処理"""

    file_name = csv_file.name

    created_at = extract_datetime_from_filename(
        file_name
    )

    file_kind = (
        "A"
        if "アナログ" in file_name
        else "D"
    )

    compare_key_class = (
        "COMPARE_KEY_A"
        if file_kind == "A"
        else "COMPARE_KEY_D"
    )

    compare_keys = get_compare_keys(
        conn,
        compare_key_class
    )

    header_table, version_table, latest_view = \
        resolve_object_names(
            folder_name,
            file_kind
        )

    csv_records = read_header_csv(
        csv_file,
        file_kind
    )

    latest_version = get_latest_version(
        conn,
        version_table,
        ship_no
    )

    # 初回取込
    if latest_version is None:

        logger.info("Initial Import")

        insert_version(
            conn,
            version_table,
            ship_no,
            0,
            created_at
        )

        seq_no = 1

        for record in csv_records:

            insert_header(
                conn,
                header_table,
                record,
                ship_no,
                0,
                seq_no,
                None,
                file_kind
            )

            seq_no += 1

        return

    # Latestロード
    latest_records = load_latest_headers(
        conn,
        latest_view,
        ship_no,
        compare_keys
    )

    csv_dict = {}

    for record in csv_records:

        csv_dict[
            build_key(
                record,
                compare_keys
            )
        ] = record

    csv_keys = set(csv_dict.keys())
    latest_keys = set(latest_records.keys())

    add_keys = csv_keys - latest_keys
    delete_keys = latest_keys - csv_keys

    # 完全一致
    if len(add_keys) == 0 and len(delete_keys) == 0:

        logger.info("No Difference")
        return

    # 差分あり
    new_version = latest_version + 1

    logger.info(f"CurrentVersion={latest_version}")
    logger.info(f"NewVersion={new_version}")
    logger.info(f"AddCount={len(add_keys)}")
    logger.info(f"DeleteCount={len(delete_keys)}")

    insert_version(
        conn,
        version_table,
        ship_no,
        new_version,
        created_at
    )

    seq_no = 1

    # 追加
    for key in sorted(add_keys):

        insert_header(
            conn,
            header_table,
            csv_dict[key],
            ship_no,
            new_version,
            seq_no,
            "A",
            file_kind
        )

        seq_no += 1

    # 削除
    for key in sorted(delete_keys):

        insert_header(
            conn,
            header_table,
            latest_records[key],
            ship_no,
            new_version,
            seq_no,
            "D",
            file_kind
        )

        seq_no += 1


def run(
    engine,
    ship_no,
    config
):
    """UNTEN全体処理"""

    with engine.connect() as conn:

        folder_infos = get_target_folders(
            conn,
            config["variables"]["unten_folder_l_class"],
            config["variables"]["unten_folder_m_class"]
        )

    file_count = 0

    for folder_name, folder_path in folder_infos:

        logger.info(f"Folder={folder_name}")

        with engine.connect() as conn:

            target_patterns = get_target_file_patterns(
                conn,
                folder_name
            )

        logger.info(
            f"target_patterns={target_patterns}"
        )

        csv_files = []

        for csv_file in Path(folder_path).glob("*.csv"):

            for pattern in target_patterns:

                if pattern in csv_file.name:

                    csv_files.append(csv_file)
                    break

        csv_files.sort()

        logger.info(
            f"target_csv_count={len(csv_files)}"
        )

        for csv_file in csv_files:

            logger.info(f"[START] {csv_file.name}")

            try:

                with engine.begin() as conn:

                    process_csv_file(
                        conn,
                        ship_no,
                        folder_name,
                        csv_file
                    )

                file_count += 1

                logger.info(f"[END] {csv_file.name}")

            except Exception as ex:

                logger.error(csv_file.name)
                logger.exception(str(ex))

                continue

    logger.info(f"読取ファイル数:{file_count}")






変換.txt

   Unnamed: 0           Unnamed: 1            Unnamed: 2                           Unnamed: 3                                          Unnamed: 4
           項目 1つ目（FFMLOG CSVヘッダ取込）                   NaN                      2つ目（InfluxDB取込）                                                 NaN
        ファイル名              main.py                   NaN                     influx_import.py                                                 NaN
         主な目的       CSVヘッダ情報をDBへ登録                   NaN                   CSVデータをInfluxDBへ投入                                                 NaN
         実行引数                   なし                   NaN                            UNTEN MGM                                                 NaN
         設定読込        load_config()   modules/settings.py load_config() , load_import_config() modules/settings.py\n＊load_import_config()はこのファイル自身
         DB接続   create_db_engine()         modules/db.py                   create_db_engine()                                       modules/db.py
m_variables参照                   あり                   NaN                                   なし                                                 NaN
      m_ffm参照                  間接的                   NaN                                 直接参照                                                 NaN
      艦番号取得方法        get_ship_no() modules/repository.py               get_ship_no_by_ryaku()                               modules/repository.py
  FILE_TYPE取得      get_file_type() modules/repository.py                                   なし                                                 NaN
      単位マスタ取得                   なし                   NaN                      get_unit_dict()                               modules/repository.py
      ヘッダ変換取得    dispatcher内の可能性あり                   NaN               get_header_tag_cache()                               modules/repository.py
Measurement使用                   なし                   NaN                              あり（MGM）                                                 NaN
   InfluxDB書込                   なし                   NaN                     write_influxdb()                                                 NaN
        CSV読込          dispatcher側                   NaN                       process_file()                                                 NaN
         最終処理                run() modules/dispatcher.py                       process_file()                                                 NaN





変換（txt→csv）.py

import pandas as pd
from pathlib import Path

excel_file = next(Path(".").glob("*.xlsx"))

# すべて文字列として読込
df = pd.read_excel(excel_file, dtype=str)

# TXT出力
df.to_csv("変換.txt", index=False)

# CSV出力
df.to_csv("変換.csv", index=False, encoding="utf-8-sig")

print("変換.txt と 変換.csv を作成しました")








         











